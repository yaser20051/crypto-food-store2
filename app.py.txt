from flask import Flask, request, jsonify, render_template, redirect
import requests
import time

app = Flask(__name__, template_folder="templates")

# بيانات المحفظة ومفتاح API
STORE_WALLET_ADDRESS = "TPBv2PYACGBmz1XtPQBzYoNfGEZCnKuJVT"
USDT_CONTRACT_ADDRESS = "TXLAQ63Xg1NAzckPwKHvzw7CSEmLMEqcdj"
TRONGRID_API_KEY = "6d27e8c3-7d0d-4c6f-88df-f7c2d5a2c17b"

# تخزين الطلبات مؤقتاً في الذاكرة
orders = {}

@app.route("/")
def home():
    return render_template("index.html")

@app.route("/create-order", methods=["POST"])
def create_order():
    data = request.json
    order_id = str(int(time.time()))
    amount = data.get("amount")
    user_wallet = data.get("wallet")

    orders[order_id] = {
        "amount": amount,
        "wallet": user_wallet,
        "paid": False,
        "tx_id": None,
        "notified": False,
        "created_at": time.time()
    }

    return jsonify({
        "message": "Order created",
        "order_id": order_id,
        "store_wallet": STORE_WALLET_ADDRESS,
        "amount": amount
    })

# صفحة متابعة الطلب مع عرض التفاصيل والعداد التنازلي
@app.route("/order/<order_id>")
def order_page(order_id):
    if order_id not in orders:
        return "طلب غير موجود", 404
    order = orders[order_id]
    return render_template("order.html",
                           order_id=order_id,
                           amount=order["amount"],
                           store_wallet=STORE_WALLET_ADDRESS)

# التحقق من حالة الدفع
@app.route("/check-payment/<order_id>", methods=["GET"])
def check_payment(order_id):
    if order_id not in orders:
        return jsonify({"error": "Invalid order ID"}), 404

    order = orders[order_id]
    amount = order["amount"]
    user_wallet = order["wallet"]

    headers = {
        "accept": "application/json",
        "TRON-PRO-API-KEY": TRONGRID_API_KEY
    }

    url = f"https://api.trongrid.io/v1/accounts/{STORE_WALLET_ADDRESS}/transactions/trc20?limit=50&contract_address={USDT_CONTRACT_ADDRESS}"
    response = requests.get(url, headers=headers)
    txs = response.json().get("data", [])

    for tx in txs:
        if (tx["from"] == user_wallet and
            tx["to"] == STORE_WALLET_ADDRESS and
            float(tx["value"]) / (10 ** 6) >= float(amount)):
            orders[order_id]["paid"] = True
            orders[order_id]["tx_id"] = tx["transaction_id"]

            if not orders[order_id]["notified"]:
                print(f"🔔 إشعار: تم الدفع للطلب {order_id} | TX: {tx['transaction_id']}")
                orders[order_id]["notified"] = True

            return jsonify({"paid": True, "tx_id": tx["transaction_id"]})

    # تحقق هل الوقت تجاوز 5 دقائق (300 ثانية)
    elapsed = time.time() - order["created_at"]
    if elapsed > 300 and not orders[order_id]["paid"]:
        return redirect("/fail")

    return jsonify({"paid": False})

@app.route("/success")
def payment_success():
    return render_template("success.html")

@app.route("/fail")
def payment_fail():
    return render_template("fail.html")

@app.route("/orders", methods=["GET"])
def list_orders():
    return jsonify(orders)

@app.route("/admin")
def admin_panel():
    html = """
    <html>
    <head><title>لوحة الإدارة</title></head>
    <body>
        <h2>جميع الطلبات</h2>
        <table border="1" cellpadding="5">
            <tr>
                <th>رقم الطلب</th>
                <th>المبلغ</th>
                <th>المحفظة</th>
                <th>تم الدفع</th>
                <th>TX ID</th>
            </tr>
    """
    for order_id, data in orders.items():
        html += f"""
            <tr>
                <td>{order_id}</td>
                <td>{data['amount']}</td>
                <td>{data['wallet']}</td>
                <td>{'✅' if data['paid'] else '❌'}</td>
                <td>{data['tx_id'] or '-'}</td>
            </tr>
        """
    html += """
        </table>
    </body>
    </html>
    """
    return html

if __name__ == "__main__":
    app.run(debug=True)
