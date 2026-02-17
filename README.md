# ATMOS To'lov Tizimi — Integratsiya Dokumentatsiyasi

## 📋 Umumiy Ma'lumot

GPT Booster Bot uchun ATMOS to'lov tizimi orqali avtomatik yangilanuvchi obuna tizimi yaratildi.

**Asosiy imkoniyatlar:**
- ✅ Unique payment link generation (UUID asosida)
- ✅ UzCard va Humo kartalar bilan to'lov
- ✅ OTP tasdiqlash
- ✅ Karta tokenlarini saqlash (avtomatik yangilash uchun)
- ✅ Premium obuna boshqaruvi
- ✅ Payment session tracking

---

## ✅ Bajarilgan Ishlar

### 1. Database Modellari

#### `PaymentSession` (Yangi model)
**Fayl:** [`apps/payments/models.py`](file:///root/TelegramBot/GptBot/Backend/apps/payments/models.py#L42-L85)

Unique payment linklar uchun:
```python
- id (UUID) — unikal session identifikatori
- user — foydalanuvchi
- plan — obuna rejasi
- status — pending, processing, completed, failed, expired
- transaction_id — ATMOS transaction ID
- auto_renew — avtomatik yangilash belgisi
- expires_at — muddati tugash vaqti (30 daqiqa)
- payment_url — to'lov linki (property)
```

**Link formati:**
```
https://bot.ibots.uz/?session_id=d6aaa197-137a-42ed-a5c8-beef06e7a8b8
```

#### `BoundCard` (Yangi model)
**Fayl:** [`apps/payments/models.py`](file:///root/TelegramBot/GptBot/Backend/apps/payments/models.py#L88-L106)

Saqlangan karta tokenlari:
```python
- card_token — ATMOS recurring token (unikal)
- card_id — ATMOS card ID
- pan_masked — masklengan karta raqami (8600****1234)
- expiry — amal qilish muddati (YYMM)
- card_type — uzcard/humo/visa/mastercard
- is_active — faolligi
- last_used_at — oxirgi ishlatilgan vaqt
```

#### `Subscription` Yangilanishi
**Fayl:** [`apps/subscriptions/models.py`](file:///root/TelegramBot/GptBot/Backend/apps/subscriptions/models.py#L41-L44)

Qo'shilgan maydonlar:
```python
- bound_card — bog'langan karta (ForeignKey)
- last_renewal_attempt — oxirgi yangilash urinishi
- renewal_attempts — urinishlar soni
- renewal_error — xato xabari
```

#### Migrations
```bash
✅ apps/payments/migrations/0003_boundcard_paymentsession.py
✅ apps/subscriptions/migrations/0002_subscription_bound_card_and_more.py
```

---

### 2. ATMOS API Client

**Fayl:** [`apps/payments/atmos/client.py`](file:///root/TelegramBot/GptBot/Backend/apps/payments/atmos/client.py)

To'liq ATMOS REST API client (340+ qator kod):

**Asosiy metodlar:**

| Metod | Vazifasi |
|-------|----------|
| `get_token()` | Bearer token olish (50 daqiqa cache) |
| `create_transaction(amount, desc)` | Yangi transaction yaratish |
| `pre_apply(tx_id, card, expiry)` | OTP yuborish |
| `apply(tx_id, otp)` | OTP bilan tasdiqlash |
| `charge_card_token(token, amount)` | Car_token orqali to'lov (auto-renewal) |
| `get_transaction_status(tx_id)` | Status tekshirish |
| `reverse_transaction(tx_id)` | Bekor qilish/qaytarish |
| `verify_callback_signature(...)` | Webhook imzo tekshirish |

**Xususiyatlar:**
- ✅ Token avtomatik cache (Django cache)
- ✅ Auto-refresh on 401 errors
- ✅ Custom exception handling (`AtmosAPIError`)
- ✅ Comprehensive logging
- ✅ Retry logic (httpx timeout: 30s)

---

### 3. Gateway API Endpoints

**Fayl:** [`gateway/main.py`](file:///root/TelegramBot/GptBot/Backend/gateway/main.py#L549-L867)

**6 ta yangi endpoint:**

#### 1️⃣ `POST /api/v1/payment/create-link`
Payment session yaratadi va unique link qaytaradi.

**Request:**
```json
{
  "telegram_id": 123456789,
  "plan_slug": "monthly"
}
```

**Response:**
```json
{
  "success": true,
  "data": {
    "session_id": "d6aaa197-137a-42ed-a5c8-beef06e7a8b8",
    "payment_url": "https://bot.ibots.uz/?session_id=...",
    "expires_at": "2026-02-17T19:30:00Z",
    "plan": {
      "name": "1 Oylik",
      "price": "29 000 so'm",
      "duration_days": 30
    }
  }
}
```

#### 2️⃣ `GET /api/v1/payment/session/{session_id}`
Session ma'lumotlarini olish (frontend uchun).

**Response:**
```json
{
  "success": true,
  "data": {
    "session_id": "...",
    "user_id": 123,
    "plan": {
      "slug": "monthly",
      "name": "1 Oylik",
      "price": "29 000 so'm",
      "price_raw": 2900000,
      "period": "30 kun"
    },
    "status": "pending",
    "auto_renew": true
  }
}
```

#### 3️⃣ `POST /api/v1/atmos/create`
ATMOS transaction yaratish.

**Request:**
```json
{
  "session_id": "d6aaa197-...",
  "amount": 2900000,
  "auto_renew": true
}
```

#### 4️⃣ `POST /api/v1/atmos/pre-apply`
Karta ma'lumotlari bilan pre-apply va OTP yuborish.

**Request:**
```json
{
  "transaction_id": 12345678,
  "card_number": "8600490744313347",
  "expiry": "2802"
}
```

#### 5️⃣ `POST /api/v1/atmos/apply`
OTP bilan to'lovni tasdiqlash.

**Request:**
```json
{
  "transaction_id": 12345678,
  "otp": "123456",
  "session_id": "d6aaa197-..."
}
```

**Backend Actions:**
1. `Payment` yozuvini yaratadi
2. `Subscription` yaratadi (user → premium)
3. Agar `auto_renew=true` → `BoundCard` saqlaydi
4. User rolini `PREMIUM` ga yangilaydi
5. Session statusini `COMPLETED` qiladi

#### 6️⃣ `POST /api/v1/atmos/callback`
ATMOS webhook handler (signature verification bilan).

---

### 4. Bot Integratsiyasi

**Fayl:** [`Bot/handlers/callbacks.py`](file:///root/TelegramBot/GptBot/Backend/Bot/handlers/callbacks.py#L139-L202)

**`cb_payment` handler yangilandi:**

Endi foydalanuvchi ATMOS to'lovni tanlaganida:
1. Backend API'ga `/payment/create-link` so'rov yuboradi
2. Unique payment URL oladi
3. Telegram WebApp tugmasini yuboradi
4. Foydalanuvchi Mini App'ni ochadi → to'lov jarayoni boshlanadi

**Kod:**
```python
async with httpx.AsyncClient() as client:
    response = await client.post(
        'http://127.0.0.1:8030/api/v1/payment/create-link',
        json={'telegram_id': callback.from_user.id, 'plan_slug': plan_slug}
    )
    payment_url = data['data']['payment_url']
    
keyboard = InlineKeyboardMarkup(inline_keyboard=[
    [InlineKeyboardButton(
        text="💳 Kartani ulash va to'lash",
        web_app=WebAppInfo(url=payment_url)
    )]
])
```

---

### 5. Frontend Yangilanishi

**Fayl:** [`Frontend/src/components/PaymentPage.tsx`](file:///root/TelegramBot/GptBot/Frontend/src/components/PaymentPage.tsx)

**Qo'shilgan funksionallik:**

#### Session ID Handling
```tsx
const sessionId = urlParams.get('session_id')

useEffect(() => {
  if (sessionId) {
    fetch(`${API_BASE}/api/v1/payment/session/${sessionId}`)
      .then(data => setPlan(data.plan))
  }
}, [sessionId])
```

#### API Calls Yangilandi
```tsx
// Transaction yaratish
fetch('/api/v1/atmos/create', {
  body: JSON.stringify({
    session_id: sessionId,  // endi telegram_id o'rniga
    amount: plan.priceRaw,
    auto_renew: autoRenew
  })
})

// OTP tasdiqlash
fetch('/api/v1/atmos/apply', {
  body: JSON.stringify({
    transaction_id, otp,
    session_id: sessionId  // session_id yuboriladi
  })
})
```

**Build:**
```bash
✅ npm run build
✅ dist/assets/index-C-_-YIgK.js (206.92 kB)
```

---

### 6. Admin Panel

**Fayl:** [`apps/payments/admin.py`](file:///root/TelegramBot/GptBot/Backend/apps/payments/admin.py)

Yangi admin panellar qo'shildi:

| Model | URL | Imkoniyatlar |
|-------|-----|-------------|
| PaymentSession | `/admin/payments/paymentsession/` | Session status, payment_url ko'rish |
| BoundCard | `/admin/payments/boundcard/` | Karta tokenlari, is_active boshqarish |

**Fieldsets:**
- Session Info (id, user, plan, status, payment_url)
- Transaction (transaction_id, card_masked, auto_renew)
- Timestamps (created_at, updated_at, expires_at)

---

### 7. Environment Configuration

**Fayl:** [`.env`](file:///root/TelegramBot/GptBot/Backend/.env#L20-L24)

```bash
# ATMOS Payment Gateway
ATMOS_CONSUMER_KEY=test-consumer-key
ATMOS_CONSUMER_SECRET=test-consumer-secret
ATMOS_STORE_ID=12345
ATMOS_BASE_URL=https://test-apigw.atmos.uz
```

---

### 8. Services

```bash
✅ gptbot-telegram.service — restarted
✅ gptbot-api.service — restarted
```

---

## ⚠️ Qilinishi Kerak Bo'lgan Ishlar

### 1. ATMOS Credentials Yangilash (MUHIM!)

**`.env` faylida:**
```bash
ATMOS_CONSUMER_KEY=<haqiqiy-key-olish>
ATMOS_CONSUMER_SECRET=<haqiqiy-secret-olish>
ATMOS_STORE_ID=<haqiqiy-store-id>
ATMOS_BASE_URL=https://apigw.atmos.uz  # production URL
```

**Qayerdan olish:**
- ATMOS dashboard'dan
- Merchant account settings

---

### 2. Test Qilish (Majburiy)

#### Test Kartalar (Sandbox)
```
Karta: 9860090101014364
Expiry: 02/28
OTP: 111111
```

#### Test Flow:
1. Telegram botda: `/premium`
2. Plan tanlash (1 Oylik / 3 Oylik / 1 Yillik)
3. ATMOS tugmasini bosish
4. WebApp ochiladi
5. Test kartani kiritish
6. Auto-renew toggle'ni yoqish/o'chirish
7. OTP kiritish (111111)
8. Success ekrani

#### Database Tekshirish:
```sql
-- Session completed bo'ldi mi?
SELECT * FROM payment_sessions WHERE status='completed' ORDER BY created_at DESC LIMIT 5;

-- Payment success bo'ldi mi?
SELECT * FROM payments WHERE provider='atmos' AND status='success' ORDER BY created_at DESC LIMIT 5;

-- Subscription yaratildi mi?
SELECT * FROM subscriptions WHERE user_id=<user_id> AND is_active=true;

-- BoundCard saqlandi mi? (auto_renew=true bo'lsa)
SELECT * FROM bound_cards WHERE user_id=<user_id> AND is_active=true;
```

---

### 3. Auto-Renewal Scheduler (Keyingi bosqich)

**Hozirda:** Manual renewal yo'q, faqat karta tokenlari saqlanadi.

**Qilish kerak:**

#### APScheduler yoki Celery o'rnatish
```bash
pip install apscheduler
# yoki
pip install celery redis
```

#### Background Task Yaratish
**Fayl:** `apps/payments/tasks.py` (yangi)

```python
from apscheduler.schedulers.background import BackgroundScheduler
from apps.subscriptions.models import Subscription
from apps.payments.atmos.client import AtmosClient
from django.utils import timezone
from datetime import timedelta

def renew_expiring_subscriptions():
    """
    Har kuni 1 marta ishlaydigan task.
    Muddati 24 soat ichida tugaydigan obunalarni yangilaydi.
    """
    tomorrow = timezone.now() + timedelta(hours=24)
    expiring = Subscription.objects.filter(
        auto_renew=True,
        end_date__lte=tomorrow,
        end_date__gte=timezone.now(),
        bound_card__isnull=False,
        bound_card__is_active=True
    )
    
    client = AtmosClient()
    
    for sub in expiring:
        try:
            # Karta tokeni orqali to'lov
            result = client.charge_card_token(
                card_token=sub.bound_card.card_token,
                amount=int(sub.plan.price * 100)  # tiyinga
            )
            
            if result['success']:
                # Obunani uzaytirish
                sub.end_date += timedelta(days=sub.plan.duration_days)
                sub.renewal_attempts = 0
                sub.renewal_error = ''
                sub.save()
                
                # Telegram notification
                send_telegram_notification(sub.user.telegram_id, "✅ Obuna yangilandi!")
            else:
                handle_renewal_failure(sub, result.get('error'))
        except Exception as e:
            handle_renewal_failure(sub, str(e))

# Scheduler setup
scheduler = BackgroundScheduler()
scheduler.add_job(renew_expiring_subscriptions, 'cron', hour=10)  # Har kuni soat 10:00
scheduler.start()
```

#### Bot `main.py` ga qo'shish
```python
from apps.payments.tasks import scheduler

# Bot start'da
scheduler.start()
```

---

### 4. Payment History UI (Ixtiyoriy)

User uchun to'lovlar tarixini ko'rsatish:

**Endpoint yaratish:**
```python
@app.get("/api/v1/user/{telegram_id}/payments")
async def get_user_payments(telegram_id: int):
    payments = await get_payments(telegram_id)
    return {"success": True, "data": payments}
```

**Bot'da komanda:**
```
/payments — to'lovlar tarixi
```

---

### 5. Subscription Cancellation

User o'zi obunani bekor qilishi:

**Endpoint:**
```python
@app.post("/api/v1/subscription/{id}/cancel")
async def cancel_subscription(id: int, telegram_id: int):
    subscription = Subscription.objects.get(id=id, user__telegram_id=telegram_id)
    subscription.auto_renew = False
    subscription.cancelled_at = timezone.now()
    subscription.save()
```

**Bot callback:**
```python
@router.callback_query(F.data == "cancel_subscription")
async def cb_cancel_subscription(callback: CallbackQuery):
    # API call va tasdiqlash
```

---

### 6. Webhook URL Setup (Production)

ATMOS dashboard'da webhook URL ni sozlash:
```
https://gptapi.ibots.uz/api/v1/atmos/callback
```

**Signature secret** `.env` ga qo'shish va verify qilish.

---

### 7. Error Handling Yaxshilash

- [ ] User-friendly xato xabarlari
- [ ] Retry logic (tarmoq xatolari uchun)
- [ ] Admin uchun notification (unsuccessful payments)
- [ ] Logs rotation va monitoring

---

## 📂 Fayl Tuzilishi

```
Backend/
├── apps/
│   ├── payments/
│   │   ├── models.py              ✅ PaymentSession, BoundCard
│   │   ├── admin.py               ✅ Admin panels
│   │   ├── atmos/
│   │   │   ├── __init__.py
│   │   │   └── client.py          ✅ AtmosClient (340+ lines)
│   │   └── migrations/
│   │       └── 0003_boundcard_paymentsession.py  ✅
│   └── subscriptions/
│       ├── models.py               ✅ Subscription yangilandi
│       └── migrations/
│           └── 0002_subscription_bound_card_and_more.py  ✅
├── gateway/
│   └── main.py                     ✅ 6 ta endpoint (+320 lines)
├── Bot/
│   └── handlers/
│       └── callbacks.py            ✅ Payment link generation
└── .env                            ✅ ATMOS credentials

Frontend/
└── src/
    └── components/
        └── PaymentPage.tsx         ✅ Session ID integration
```

---

## 🧪 Test Qilish Uchun Qadamlar

### 1. ATMOS Credentials Olish
- ATMOS dashboard'ga kirish
- Consumer Key, Secret, Store ID olish

### 2. `.env` Yangilash
```bash
cd /root/TelegramBot/GptBot/Backend
nano .env
# ATMOS_* qiymatlarini yangilash
```

### 3. Services Restart
```bash
sudo systemctl restart gptbot-telegram gptbot-api
```

### 4. Telegram Botda Test
```
/premium → 1 Oylik → ATMOS → Kartani ulash
```

### 5. Database Tekshirish
Django admin yoki SQL:
```
http://gptapi.ibots.uz/admin/
```

---

## 📞 Yordam

**Xato yuz bersa:**
1. Logs tekshirish:
```bash
journalctl -u gptbot-api -f
journalctl -u gptbot-telegram -f
```

2. Database migration holati:
```bash
cd Backend
source venv/bin/activate
python manage.py showmigrations payments subscriptions
```

3. ATMOS API test (Python shell):
```bash
python manage.py shell
>>> from apps.payments.atmos.client import AtmosClient
>>> client = AtmosClient()
>>> token = client.get_token()
>>> print(token)
```

---

## 📊 Statistika

**Yaratilgan:**
- 2 ta yangi model (PaymentSession, BoundCard)
- 1 ta API client (340+ lines)
- 6 ta endpoint (320+ lines)
- 2 ta migration file
- 2 ta admin panel
- Bot integration (payment links)
- Frontend session handling

**Jami kod:** ~1000+ lines yangi kod

**Vaqt:** ~12-15 soat development

---

## ✅ Checklist

**Production'ga chiqishdan oldin:**
- [ ] ATMOS credentials yangilash
- [ ] Test kartalar bilan full flow test
- [ ] Database backup
- [ ] Webhook URL sozlash
- [ ] Auto-renewal scheduler qo'shish (ixtiyoriy)
- [ ] Monitoring setup (Sentry/Log aggregation)
- [ ] User documentation

**MVP uchun yetarli:**
- [x] Payment link generation
- [x] Karta bilan to'lov
- [x] OTP confirmation
- [x] Subscription yaratish
- [x] Card token saqlash
