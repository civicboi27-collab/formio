# FormIO Survey — Deployment Guide

## Tech Stack

| Layer        | Technology               | Version    |
|--------------|--------------------------|------------|
| **Frontend** | React                    | 18.3.x     |
| **Frontend** | TypeScript               | 5.6.x      |
| **Frontend** | Vite (bundler)           | 5.4.x      |
| **Form Engine** | formiojs / @formio/react | 4.21.x / 5.3.x |
| **UI**       | Bootstrap                | 5.3.x      |
| **Routing**  | React Router DOM         | 6.28.x     |
| **Backend**  | Node.js + Express        | 5.2.x      |
| **Database** | MySQL                    | 8.x        |
| **ORM/Driver** | mysql2/promise          | 3.18.x     |
| **Runtime**  | Node.js                  | ≥ 18.x     |

---

## โครงสร้างโปรเจ็ค

```
formio-survey/
├── dist/                   ← build output (สร้างโดย npm run build)
├── public/
├── src/                    ← React frontend
│   ├── pages/
│   ├── components/
│   ├── store/
│   ├── types/
│   └── i18n/
├── server/
│   ├── index.js            ← Express API server (port 3001)
│   ├── schema.sql          ← สร้าง DB tables
│   ├── seed.js             ← seed จาก db.json
│   ├── seed-exam.js        ← seed ข้อสอบตัวอย่าง
│   └── seed-activity.js    ← seed แบบประเมินกิจกรรม
├── .env                    ← environment variables (ต้องสร้างเอง)
├── package.json
└── vite.config.ts
```

---

## Environment Variables

สร้างไฟล์ `.env` ที่ root ของโปรเจ็ค:

```env
# Server port
PORT=3001

# MySQL connection
DB_HOST=localhost
DB_PORT=3306
DB_USER=root
DB_PASS=your_password_here
DB_NAME=formio_survey
```

---

## การ Deploy บน Server (Production)

### 1. ติดตั้ง dependencies บน server

```bash
# ต้องมี Node.js >= 18 และ npm
node -v
npm -v

# ติดตั้ง packages
npm install
```

### 2. ตั้งค่า MySQL Database

```bash
# สร้าง database (หรือให้ server auto-create เมื่อ start)
mysql -u root -p < server/schema.sql
```

> **หมายเหตุ:** server/index.js จะ auto-create tables เมื่อ start — ไม่จำเป็นต้องรัน schema.sql ก็ได้

### 3. สร้าง .env ไฟล์

```bash
cp .env.example .env
# แก้ไขค่าให้ตรงกับ production server
nano .env
```

### 4. Build Frontend

```bash
npm run build
# output อยู่ที่ dist/
```

### 5. Start Server

Express จะ serve React build (`dist/`) โดยอัตโนมัติเมื่อมีโฟลเดอร์ `dist/` อยู่

```bash
node server/index.js
# หรือ
npm start
```

> ทั้ง frontend และ API จะรันอยู่บน **port 3001** port เดียว

---

## การ Deploy ด้วย PM2 (แนะนำสำหรับ Production)

```bash
# ติดตั้ง PM2
npm install -g pm2

# Start
pm2 start server/index.js --name formio-survey

# Auto-start เมื่อ server reboot
pm2 startup
pm2 save

# ดู logs
pm2 logs formio-survey

# Restart
pm2 restart formio-survey
```

---

## การ Deploy ด้วย Docker

**Dockerfile** (สร้างเองถ้าต้องการ):

```dockerfile
FROM node:20-alpine
WORKDIR /app
COPY package*.json ./
RUN npm ci --omit=dev
COPY . .
RUN npm run build
EXPOSE 3001
CMD ["node", "server/index.js"]
```

```bash
docker build -t formio-survey .
docker run -d \
  -p 3001:3001 \
  -e DB_HOST=your_db_host \
  -e DB_USER=your_db_user \
  -e DB_PASS=your_db_pass \
  -e DB_NAME=formio_survey \
  --name formio-survey \
  formio-survey
```

---

## Nginx Reverse Proxy (ถ้า expose port 80/443)

```nginx
server {
    listen 80;
    server_name yourdomain.com;

    location / {
        proxy_pass         http://localhost:3001;
        proxy_http_version 1.1;
        proxy_set_header   Upgrade $http_upgrade;
        proxy_set_header   Connection 'upgrade';
        proxy_set_header   Host $host;
        proxy_cache_bypass $http_upgrade;
    }
}
```

---

## API Endpoints

| Method | Path                          | Description            |
|--------|-------------------------------|------------------------|
| GET    | `/api/forms`                  | ดึงฟอร์มทั้งหมด        |
| POST   | `/api/forms`                  | สร้างฟอร์มใหม่         |
| GET    | `/api/forms/:id`              | ดึงฟอร์มตาม ID         |
| PUT    | `/api/forms/:id`              | แก้ไขฟอร์ม             |
| DELETE | `/api/forms/:id`              | ลบฟอร์ม                |
| GET    | `/api/forms/:id/responses`    | ดึงคำตอบทั้งหมดของฟอร์ม |
| POST   | `/api/responses`              | บันทึกคำตอบ            |

---

## Scripts

| คำสั่ง                     | ทำอะไร                                      |
|----------------------------|---------------------------------------------|
| `npm run dev`              | รัน Vite dev server + Express พร้อมกัน      |
| `npm run build`            | Build React frontend → `dist/`              |
| `npm start`                | รัน Express (production)                    |
| `npm run build:start`      | Build แล้ว start ทันที                      |
| `node server/seed-exam.js` | Seed ข้อสอบตัวอย่าง (วิทย์ ป.4, JS)        |
| `node server/seed-activity.js` | Seed แบบประเมินผลการเข้าร่วมกิจกรรม    |

---

## Production Checklist

- [ ] ตั้งค่า `.env` ครบทุกตัวแปร
- [ ] MySQL database พร้อมใช้งาน
- [ ] รัน `npm run build` สำเร็จ
- [ ] folder `dist/` มีอยู่ก่อน start server
- [ ] เปิด port 3001 (หรือ port ที่กำหนด) ใน firewall
- [ ] ตั้งค่า PM2 หรือ process manager
- [ ] ตั้งค่า Nginx reverse proxy (ถ้าต้องการ HTTPS)
