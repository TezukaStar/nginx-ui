# การติดตั้งและใช้งาน Nginx UI

## 📝 สิ่งที่ต้องเตรียม
1. Ubuntu Server (Amazon EC2)
   - Instance Type: t2.micro หรือสูงกว่า
   - Security Groups (พอร์ตที่ต้องเปิด):
     - Port 9000 (Nginx UI)
     ![alt text](amazone/4.png)

2. โดเมนและ Cloudflare
   - โดเมนที่จดทะเบียนแล้ว
   - จัดการ DNS บน Cloudflare
   - สร้าง A Record ชี้ไปที่ IP ของ EC2!

## 🚀 ขั้นตอนการติดตั้ง
#### 1. อัปเดตและอัปเกรดระบบ
```bash
# อัปเดตรายการแพ็คเกจ
sudo apt update

# อัปเกรดแพ็คเกจทั้งหมด
sudo apt upgrade -y
```

#### 2. ติดตั้ง Nginx
```bash
# ติดตั้ง Nginx
sudo apt install nginx -y

# ตรวจสอบว่าติดตั้งสำเร็จ
sudo systemctl status nginx
nginx -v
```

#### 3. ติดตั้ง Nginx UI
```bash
# ติดตั้ง Nginx UI
sudo bash -c "$(curl -L https://raw.githubusercontent.com/0xJacky/nginx-ui/main/install.sh)" @ install

# เปิดใช้งานและเริ่มต้นบริการ
sudo systemctl enable nginx-ui
sudo systemctl start nginx-ui

# เข้าเว็บ Nginx UI
http://<your-ip>:9000
```

#### 4. ติดตั้งสภาพแวดล้อมสำหรับ Node.js
```bash
# ติดตั้ง Node.js
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt install -y nodejs

node -v
npm -v

# ติดตั้ง PM2 สำหรับจัดการโปรเซส
sudo npm install -g pm2

pm2 -v
```


## ⚙️ การตั้งค่าระบบ

### 1. ตั้งค่า SSL Certificate

1. **ตั้งค่า DNS Credentials**
   - เข้าเว็บ Nginx UI: `http://<your-ip>:9000`
   - ไปที่: Certificates > DNS Credentials
   - เพิ่มข้อมูล Cloudflare:
     - API Token
     - อีเมล

2. **สร้าง Certificate**
   - ไปที่: Certificates > Certificates List
   - สร้าง Issue Wildcard Certificate ใหม่
   - กรอกข้อมูลโดเมน
   - เลือก Cloudflare เป็น DNS Provider
   - สร้าง symbolic link กับตำแหน่งไฟล์ certification
   ```bash
   # สร้างโฟลเดอร์ dev 
   sudo mkdir -p /etc/nginx/ssl/dev

   # สร้าง symbolic link สำหรับไฟล์ใบรับรอง
   sudo ln -sf /etc/nginx/ssl/*.<domain>_2048/fullchain.cer /etc/nginx/ssl/dev/fullchain.pem
   sudo ln -sf /etc/nginx/ssl/*.<domain>_2048/private.key /etc/nginx/ssl/dev/privkey.pem
   
   #ตรวจสอบว่า symbolic link ถูกสร้างขึ้นอย่างถูกต้อง
   ls -la /etc/nginx/ssl/dev/
   ```

### 2. ตั้งค่า Nginx

1. **สร้างไฟล์คอนฟิกสำหรับแอพ**
   - ไปที่: Manage Configs
   - สร้างไฟล์ใหม่: `nodeapp.conf`
   ```nginx
    server {
        listen 80;
        server_name <domain>;

        location / {
            if ($http_cf_visitor ~ '{"scheme":"https"}') {
                proxy_pass http://localhost:3000;
                break;
            }
            return 301 https://$host$request_uri;
            }
    }

    server {
        listen 443 ssl;
        server_name <domain>;
        
        ssl_certificate /etc/nginx/ssl/dev/fullchain.pem;
        ssl_certificate_key /etc/nginx/ssl/dev/privkey.pem;

        location / {
            proxy_pass http://localhost:3000;
            proxy_http_version 1.1;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection 'upgrade';
            proxy_set_header Host $host;
            proxy_cache_bypass $http_upgrade;
        }
   }
   ```

2. **อัพเดทไฟล์คอนฟิกหลัก**
   - แก้ไขไฟล์: `nginx.conf`
   - เพิ่มในส่วน http:
   ```nginx
   include /etc/nginx/nodeapp.conf;
   ```

## 🎮 การเริ่มใช้งาน

1. **รันแอพพลิเคชัน Node.js**
```bash
# เตรียมโฟลเดอร์โปรเจค
mkdir nodeapp
cd nodeapp

npm init -y

npm install express

sudo nano server.js

# โค้ด Back-end
const express = require('express');
const app = express();
const port = 3000;

app.get('/', (req, res) => {
  res.json({ message: 'Backend is working!' });
});

app.listen(port, () => {
  console.log(`Server running at http://localhost:${port}`);
});

# รัน process nodeapp
pm2 start server.js

# ดู process ทั้งหมด
pm2 list

# ทดสอบ localhost
curl http://localhost:3000

curl https:<ip_address>

curl https://<domain>
```

2. **ตรวจสอบสถานะบริการ**
```bash
# เช็คสถานะ Nginx
sudo systemctl status nginx

# เช็คสถานะ Nginx UI
sudo systemctl status nginx-ui

# เช็คสถานะโปรเซส PM2
pm2 status
```