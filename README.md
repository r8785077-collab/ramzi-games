import os, secrets, requests
from datetime import datetime
from flask import Flask, request, redirect, url_for, session, flash, render_template_string, abort
from flask_sqlalchemy import SQLAlchemy

app = Flask(__name__)
app.secret_key = os.environ.get("SECRET_KEY", "CHANGE_THIS_SECRET_BEFORE_DEPLOYING")
db_url = os.environ.get("DATABASE_URL", "sqlite:///ramzi_games.db")
if db_url.startswith("postgres://"):
    db_url = db_url.replace("postgres://", "postgresql+psycopg://", 1)
elif db_url.startswith("postgresql://"):
    db_url = db_url.replace("postgresql://", "postgresql+psycopg://", 1)
app.config["SQLALCHEMY_DATABASE_URI"] = db_url
app.config["SQLALCHEMY_TRACK_MODIFICATIONS"] = False
app.config["MAX_CONTENT_LENGTH"] = 2 * 1024 * 1024
db = SQLAlchemy(app)

BOT_TOKEN = os.environ.get("BOT_TOKEN", "")
BOT_USERNAME = os.environ.get("BOT_USERNAME", "Hhbhjbbhhbot")
ADMIN_CHAT_ID = os.environ.get("ADMIN_CHAT_ID", "")
WEBHOOK_SECRET = os.environ.get("WEBHOOK_SECRET", "")
ADMIN_PASSWORD = os.environ.get("ADMIN_PASSWORD", "")
STORE_URL = os.environ.get("STORE_URL", "").rstrip("/")
CCP_NAME = os.environ.get("CCP_NAME", "اسم صاحب الحساب")
CCP_NUMBER = os.environ.get("CCP_NUMBER", "أدخل رقم CCP هنا")
BARIDIMOB_NUMBER = os.environ.get("BARIDIMOB_NUMBER", "أدخل رقم الحساب هنا")
SUPPORT_TEXT = os.environ.get("SUPPORT_TEXT", "راسلنا عبر بوت Telegram")

DEFAULT_PACKS = [
    ("100 Diamonds", "100", 0),
    ("310 Diamonds", "310", 0),
    ("520 Diamonds", "520", 0),
    ("1060 Diamonds", "1060", 0),
    ("2180 Diamonds", "2180", 0),
]

class Product(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    name = db.Column(db.String(80), nullable=False)
    diamonds = db.Column(db.String(30), nullable=False)
    price_dzd = db.Column(db.Integer, nullable=False, default=0)
    active = db.Column(db.Boolean, default=True)

class Order(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    code = db.Column(db.String(20), unique=True, nullable=False)
    customer_name = db.Column(db.String(100), nullable=False)
    phone = db.Column(db.String(40), nullable=False)
    player_id = db.Column(db.String(40), nullable=False)
    product_name = db.Column(db.String(80), nullable=False)
    price_dzd = db.Column(db.Integer, nullable=False)
    payment_method = db.Column(db.String(30), nullable=False)
    payment_reference = db.Column(db.String(120), nullable=False)
    note = db.Column(db.String(500), default="")
    status = db.Column(db.String(20), default="pending")
    created_at = db.Column(db.DateTime, default=datetime.utcnow)

def telegram_send(chat_id, text, keyboard=None):
    if not BOT_TOKEN or not chat_id:
        return
    payload = {"chat_id": chat_id, "text": text}
    if keyboard:
        payload["reply_markup"] = keyboard
    try:
        requests.post(f"https://api.telegram.org/bot{BOT_TOKEN}/sendMessage",
                      json=payload, timeout=12)
    except requests.RequestException:
        pass

def notify_admin(message):
    if ADMIN_CHAT_ID:
        telegram_send(ADMIN_CHAT_ID, message)

def admin_required():
    return bool(session.get("admin"))

BASE = """
<!doctype html><html lang="ar" dir="rtl"><head>
<meta charset="utf-8"><meta name="viewport" content="width=device-width,initial-scale=1">
<title>{{ title }} | RAMZI GAMES</title>
<style>
*{box-sizing:border-box}body{margin:0;background:#0b1020;color:#f5f7ff;font-family:Arial,Tahoma,sans-serif}
.wrap{max-width:900px;margin:auto;padding:18px}.brand{font-size:25px;font-weight:900;letter-spacing:1px;color:#8bc5ff}
.sub{color:#aebbd2;margin-top:4px}.panel{background:#151e32;border:1px solid #2b3b5c;border-radius:18px;padding:18px;margin:14px 0}
h1,h2{margin-top:0}.grid{display:grid;grid-template-columns:repeat(auto-fit,minmax(145px,1fr));gap:10px}
.pack{background:#202d47;border:1px solid #3b527b;border-radius:14px;padding:14px}
input,select,textarea{width:100%;padding:13px;margin:7px 0 13px;border-radius:10px;border:1px solid #405273;background:#0e1729;color:white;font-size:16px}
label{display:block;color:#d6e2f5;font-size:14px}button,.btn{display:inline-block;background:#1676d2;color:white;border:0;border-radius:11px;padding:13px 18px;font-weight:bold;text-decoration:none;cursor:pointer}
.secondary{background:#34445f}.danger{background:#9e3545}.muted{color:#aebbd2;font-size:13px}.price{color:#7ed6ff;font-weight:bold}
table{width:100%;border-collapse:collapse;font-size:14px}td,th{padding:9px;border-bottom:1px solid #34445f;text-align:right}
a{color:#8bc5ff}.notice{padding:12px;border-radius:10px;background:#243b35;margin:10px 0}.warning{background:#493b22}
@media(max-width:560px){.wrap{padding:12px}.panel{padding:14px}table{font-size:12px}}
</style></head><body><div class="wrap">
<div class="brand">💎 RAMZI GAMES</div><div class="sub">FREE FIRE TOP UP • ALGERIA</div>
{% with messages = get_flashed_messages() %}{% for m in messages %}<div class="notice">{{m}}</div>{% endfor %}{% endwith %}
{{ body|safe }}
<p class="muted">RAMZI GAMES — متجر مستقل. لا يمثل Garena أو Free Fire.</p>
</div></body></html>
"""

def page(title, body, **context):
    return render_template_string(BASE, title=title,
        body=render_template_string(body, **context))

HOME = """
<div class="panel"><h1>اشحن Diamonds في Free Fire 💎</h1>
<p class="sub">اختر الباقة، أدخل Player ID، ثم أرسل مرجع التحويل لإتمام طلبك.</p>
<div class="grid">{% for p in products %}<div class="pack">
<h3>{{p.name}}</h3><div class="price">{{ "{:,}".format(p.price_dzd) }} دج</div>
<p class="muted">{{p.diamonds}} Diamonds</p></div>{% endfor %}</div>
</div>
<div class="panel"><h2>إنشاء طلب جديد</h2>
<form method="post" action="/order">
<label>الاسم</label><input name="customer_name" required maxlength="100">
<label>رقم الهاتف</label><input name="phone" required maxlength="40" placeholder="05xxxxxxxx">
<label>Player ID الخاص باللعبة</label><input name="player_id" required maxlength="40">
<label>الباقة</label><select name="product_id" required>{% for p in products %}<option value="{{p.id}}">{{p.name}} — {{p.price_dzd}} دج</option>{% endfor %}</select>
<label>طريقة الدفع</label><select name="payment_method"><option>BaridiMob</option><option>CCP</option></select>
<label>مرجع التحويل أو رقم العملية</label><input name="payment_reference" required maxlength="120">
<label>ملاحظة (اختياري)</label><textarea name="note" rows="2" maxlength="500"></textarea>
<button type="submit">إرسال الطلب</button></form>
</div>
<div class="panel"><h2>بيانات الدفع</h2><p>صاحب الحساب: {{ccp_name}}</p>
<p>CCP: <b>{{ccp_number}}</b></p><p>BaridiMob: <b>{{baridi_number}}</b></p>
<div class="notice warning">لا ترسل كلمة المرور أو رمز OTP. لا يتم الشحن حتى نتحقق من وصول المبلغ فعلياً.</div>
<a class="btn secondary" href="https://t.me/{{bot_username}}">الدعم عبر Telegram</a>
</div>
"""

SUCCESS = """
<div class="panel"><h1>تم تسجيل طلبك ✅</h1>
<p>رقم طلبك: <b>{{order.code}}</b></p><p>الباقة: {{order.product_name}}</p>
<p>المبلغ: <b>{{order.price_dzd}} دج</b></p>
<p>الحالة: بانتظار التحقق من الدفع</p>
<p>احتفظ برقم الطلب وتواصل معنا عبر Telegram عند الحاجة.</p>
<a class="btn" href="/">العودة للمتجر</a>
</div>
"""

LOGIN = """
<div class="panel"><h2>دخول الإدارة</h2><form method="post">
<label>كلمة مرور الإدارة</label><input name="password" type="password" required>
<button>دخول</button></form></div>
"""

ADMIN = """
<div class="panel"><h2>لوحة إدارة RAMZI GAMES</h2>
<p>عدد الطلبات: {{orders|length}} | <a href="/admin/logout">تسجيل الخروج</a></p>
<table><tr><th>الطلب</th><th>الزبون</th><th>الباقة</th><th>المبلغ</th><th>الحالة</th><th>تحديث</th></tr>
{% for o in orders %}<tr><td>{{o.code}}<br><span class="muted">{{o.created_at}}</span></td>
<td>{{o.customer_name}}<br>{{o.phone}}<br>ID: {{o.player_id}}</td>
<td>{{o.product_name}}</td><td>{{o.price_dzd}} دج<br>{{o.payment_method}}<br>{{o.payment_reference}}</td>
<td>{{o.status}}</td><td><form method="post" action="/admin/status/{{o.id}}">
<select name="status"><option value="pending">قيد المراجعة</option><option value="paid">تم تأكيد الدفع</option><option value="delivered">تم الشحن</option><option value="rejected">مرفوض</option></select>
<button class="secondary">حفظ</button></form></td></tr>{% endfor %}</table>
</div>
<div class="panel"><h3>تعديل أسعار الباقات</h3>
{% for p in products %}<form method="post" action="/admin/price/{{p.id}}">
<label>{{p.name}} — السعر بالدينار</label><input type="number" min="0" name="price" value="{{p.price_dzd}}" required><button class="secondary">تحديث السعر</button></form>{% endfor %}
</div>
"""

@app.route("/")
def home():
    products = Product.query.filter_by(active=True).all()
    return page("المتجر", HOME, products=products, ccp_name=CCP_NAME,
                ccp_number=CCP_NUMBER, baridi_number=BARIDIMOB_NUMBER,
                bot_username=BOT_USERNAME)

@app.route("/order", methods=["POST"])
def create_order():
    try:
        product = Product.query.filter_by(id=int(request.form["product_id"]), active=True).first_or_404()
    except (ValueError, KeyError):
        abort(400)
    code = "RG" + secrets.token_hex(4).upper()
    order = Order(code=code,
        customer_name=request.form.get("customer_name","").strip(),
        phone=request.form.get("phone","").strip(),
        player_id=request.form.get("player_id","").strip(),
        product_name=product.name, price_dzd=product.price_dzd,
        payment_method=request.form.get("payment_method","BaridiMob"),
        payment_reference=request.form.get("payment_reference","").strip(),
        note=request.form.get("note","").strip())
    if not all([order.customer_name, order.phone, order.player_id, order.payment_reference]):
        flash("يرجى ملء جميع الحقول المطلوبة.")
        return redirect("/")
    db.session.add(order); db.session.commit()
    notify_admin(f"🛒 طلب جديد {order.code}\\nالزبون: {order.customer_name}\\nالهاتف: {order.phone}\\nPlayer ID: {order.player_id}\\nالباقة: {order.product_name}\\nالمبلغ: {order.price_dzd} دج\\nالدفع: {order.payment_method}\\nمرجع العملية: {order.payment_reference}\\nراجع لوحة الإدارة.")
    return page("تم الطلب", SUCCESS, order=order)

@app.route("/admin", methods=["GET","POST"])
def admin():
    if not admin_required():
        if request.method == "POST":
            password = request.form.get("password","")
            if ADMIN_PASSWORD and secrets.compare_digest(ADMIN_PASSWORD, password):
                session["admin"] = True
                return redirect("/admin")
            flash("كلمة المرور غير صحيحة.")
        return page("دخول الإدارة", LOGIN)
    orders = Order.query.order_by(Order.created_at.desc()).limit(200).all()
    products = Product.query.order_by(Product.id).all()
    return page("الإدارة", ADMIN, orders=orders, products=products)

@app.route("/admin/status/<int:order_id>", methods=["POST"])
def update_status(order_id):
    if not admin_required(): abort(403)
    order = Order.query.get_or_404(order_id)
    allowed = {"pending","paid","delivered","rejected"}
    new_status = request.form.get("status")
    if new_status not in allowed: abort(400)
    order.status = new_status; db.session.commit()
    flash(f"تم تحديث الطلب {order.code}.")
    return redirect("/admin")

@app.route("/admin/price/<int:product_id>", methods=["POST"])
def update_price(product_id):
    if not admin_required(): abort(403)
    product = Product.query.get_or_404(product_id)
    try: product.price_dzd = max(0, int(request.form.get("price","0")))
    except ValueError: abort(400)
    db.session.commit(); flash("تم تحديث السعر.")
    return redirect("/admin")

@app.route("/admin/logout")
def logout():
    session.clear(); return redirect("/")

@app.route("/health")
def health():
    return {"status":"ok","store":"RAMZI GAMES"}

@app.route("/telegram/webhook", methods=["POST"])
def telegram_webhook():
    if WEBHOOK_SECRET and request.headers.get("X-Telegram-Bot-Api-Secret-Token") != WEBHOOK_SECRET:
        abort(403)
    update = request.get_json(silent=True) or {}
    msg = update.get("message", {})
    chat = msg.get("chat", {})
    chat_id = chat.get("id")
    text = (msg.get("text") or "").strip()
    if not chat_id: return {"ok": True}
    if text.startswith("/id"):
        telegram_send(chat_id, f"Chat ID: {chat_id}")
    elif text.startswith("/start"):
        url = STORE_URL or "رابط المتجر غير مضبوط بعد"
        telegram_send(chat_id, "مرحبا بك في RAMZI GAMES 💎\\nلطلب شحن Free Fire افتح المتجر: " + url)
    elif text.startswith("/support"):
        telegram_send(chat_id, "أرسل رسالتك هنا وسيتم تحويلها للدعم.")
    elif text:
        if ADMIN_CHAT_ID and str(chat_id) != str(ADMIN_CHAT_ID):
            telegram_send(ADMIN_CHAT_ID, f"📩 رسالة دعم من Telegram\\nChat ID: {chat_id}\\n{text}")
            telegram_send(chat_id, "تم إرسال رسالتك للدعم. سنرد عليك عند توفرنا.")
        else:
            telegram_send(chat_id, "أهلاً بك في إدارة RAMZI GAMES.")
    return {"ok": True}

with app.app_context():
    db.create_all()
    if Product.query.count() == 0:
        for name, diamonds, price in DEFAULT_PACKS:
            db.session.add(Product(name=name, diamonds=diamonds, price_dzd=price))
        db.session.commit()

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=int(os.environ.get("PORT", 5000)))
# ramzi-games
RAMZI GAMES Store
