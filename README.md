# Target-Bot
Checks for in stock items 
import os
import requests
import smtplib
from email.mime.text import MIMEText
from time import sleep
from twilio.rest import Client

# 📍 Location to check
ZIP_CODE = "75224"

# 🃏 Product IDs (TCINs) to track from Target
PRODUCT_IDS = [
    "88164371",
    "94300067",
    "94681785",
    "88897904",
    "1003612683",
    "94681776",
    "94681784",
    "94681782"
]

# ⏱️ How often to check (seconds)
CHECK_INTERVAL = 600  # 10 minutes

# 📧 Email settings (from Railway environment variables)
SMTP_SERVER = "smtp.gmail.com"
SMTP_PORT = 587
EMAIL_USER = os.getenv("EMAIL_USER")        # Gmail address
EMAIL_PASS = os.getenv("EMAIL_PASS")        # Gmail app password
EMAIL_TO = "Jerrydeleon214@icloud.com"

# 📱 Twilio settings (from Railway environment variables)
ACCOUNT_SID = os.getenv("ACCOUNT_SID")
AUTH_TOKEN = os.getenv("AUTH_TOKEN")
MESSAGING_SERVICE_SID = os.getenv("MESSAGING_SERVICE_SID")

# List of phone numbers to notify
TO_NUMBERS = [
    "+12147145912",
    "+12149294885"
]

# Track last known state so you don’t get spammed
last_alert_status = {}

# 🔍 Check Target stock for a product
def check_stock(product_id, zip_code):
    url = (
        f"https://redsky.target.com/redsky_aggregations/v1/web/pdp_client_v1"
        f"?tcin={product_id}&key=ff457966e64d5e877fdbad070f276d3f"
        f"&has_pricing_store_id=true&zip={zip_code}"
    )
    headers = {"User-Agent": "Mozilla/5.0"}
    resp = requests.get(url, headers=headers)
    data = resp.json()

    in_stock_stores, online_in_stock, product_title = [], False, "Unknown Product"
    try:
        product_title = data["data"]["product"]["item"]["product_description"]["title"]
        availability = data["data"]["product"]["fulfillment"]["store_options"]
        for store in availability:
            store_name = store.get("location_name", "Unknown Store")
            status = store["order_pickup"]["availability_status"]
            if status == "IN_STOCK":
                in_stock_stores.append(store_name)

        shipping = data["data"]["product"]["fulfillment"]["shipping_options"]
        if shipping and shipping[0]["availability_status"] == "IN_STOCK":
            online_in_stock = True
    except Exception as e:
        print("Error parsing:", e)

    return product_title, in_stock_stores, online_in_stock

# 📧 Send email alert
def send_email(product_id, title, stores, online):
    if not EMAIL_USER or not EMAIL_PASS:
        print("⚠️ Email skipped (missing credentials).")
        return

    product_url = f"https://www.target.com/p/-/A-{product_id}"
    body = [f"{title}\n{product_url}\n"]
    if stores:
        body.append(f"In stock at stores: {', '.join(stores)}")
    if online:
        body.append("✅ Available for online shipping 🚚")
    msg = MIMEText("\n".join(body))
    msg["Subject"] = f"Target Stock Alert 🚨 ({title})"
    msg["From"] = EMAIL_USER
    msg["To"] = EMAIL_TO

    try:
        with smtplib.SMTP(SMTP_SERVER, SMTP_PORT) as server:
            server.starttls()
            server.login(EMAIL_USER, EMAIL_PASS)
            server.sendmail(EMAIL_USER, EMAIL_TO, msg.as_string())
        print("📧 Email sent!")
    except Exception as e:
        print("Email error:", e)

# 📱 Send SMS alert
def send_sms(product_id, title, stores, online):
    if not ACCOUNT_SID or not AUTH_TOKEN or not MESSAGING_SERVICE_SID:
        print("⚠️ SMS skipped (missing Twilio credentials).")
        return

    product_url = f"https://www.target.com/p/-/A-{product_id}"
    parts = [f"{title} → {product_url}"]
    if stores:
        parts.append(f"Stores: {', '.join(stores)}")
    if online:
        parts.append("✅ Online shipping 🚚")
    message_text = " | ".join(parts)

    try:
        client = Client(ACCOUNT_SID, AUTH_TOKEN)
        for number in TO_NUMBERS:   # send to each number
            message = client.messages.create(
                to=number,
                messaging_service_sid=MESSAGING_SERVICE_SID,
                body=message_text
            )
            print("📱 SMS sent to", number, "→", message.sid)
    except Exception as e:
        print("SMS error:", e)

# 🚀 Main loop
if __name__ == "__main__":
    while True:
        for pid in PRODUCT_IDS:
            title, stores, online = check_stock(pid, ZIP_CODE)
            current_status = {"stores": tuple(stores), "online": online}
            if current_status != last_alert_status.get(pid, {}):
                if stores or online:
                    send_email(pid, title, stores, online)
                    send_sms(pid, title, stores, online)
                    print(f"🚨 Alert sent: {title} ({pid}) → Stores: {stores}, Online: {online}")
                else:
                    print(f"{title} ({pid}) went OUT of stock.")
                last_alert_status[pid] = current_status
        sleep(CHECK_INTERVAL)
