#!/usr/bin/env python3
# -*- coding: utf-8 -*-
# SHOPACC THEPHI - Gộp shop + admin vào 1 file

from flask import Flask, render_template, request, jsonify, session, redirect, url_for, send_from_directory
import json, os, secrets, time, hashlib
from functools import wraps
from werkzeug.utils import secure_filename

app = Flask(__name__, template_folder="templates", static_folder="static", static_url_path="/static")
app.secret_key = secrets.token_hex(32)

DB_FILE = "database.json"
UPLOAD_FOLDER = "static/uploads"
ALLOWED_EXTENSIONS = {"png", "jpg", "jpeg", "gif", "webp"}
os.makedirs(UPLOAD_FOLDER, exist_ok=True)
app.config["UPLOAD_FOLDER"] = UPLOAD_FOLDER
app.config["MAX_CONTENT_LENGTH"] = 20 * 1024 * 1024

def allowed_file(filename):
    return "." in filename and filename.rsplit(".", 1)[1].lower() in ALLOWED_EXTENSIONS

# ========== DATABASE ==========
def load_db():
    if not os.path.exists(DB_FILE):
        return {
            "admins": {"admin": hashlib.sha256("admin123".encode()).hexdigest()},
            "accounts": {},
            "orders": {}
        }
    with open(DB_FILE, "r", encoding="utf-8") as f:
        return json.load(f)

def save_db(db):
    with open(DB_FILE, "w", encoding="utf-8") as f:
        json.dump(db, f, indent=2, ensure_ascii=False)

# ========== DECORATOR ==========
def login_required(f):
    @wraps(f)
    def wrapper(*args, **kwargs):
        if "user" not in session:
            return redirect(url_for("login"))
        return f(*args, **kwargs)
    return wrapper

# ========== SHOP (USER) ==========
@app.route("/")
def shop():
    return render_template("shop.html")

@app.route("/api/accounts", methods=["GET"])
def get_accounts_public():
    db = load_db()
    safe = {}
    for k, v in db["accounts"].items():
        safe[k] = {
            "id": k,
            "name": v.get("name"),
            "price": v.get("price"),
            "rank": v.get("rank"),
            "skin": v.get("skin"),
            "status": v.get("status", "available"),
            "images": v.get("images", [])
        }
    return jsonify(safe)

# ========== ADMIN ==========
@app.route("/login", methods=["GET", "POST"])
def login():
    if request.method == "POST":
        u = request.form.get("username")
        p = request.form.get("password")
        h = hashlib.sha256(p.encode()).hexdigest()
        db = load_db()
        if u in db["admins"] and db["admins"][u] == h:
            session["user"] = u
            return redirect(url_for("admin_panel"))
        return render_template("login.html", error="Sai tài khoản!")
    return render_template("login.html")

@app.route("/logout")
def logout():
    session.clear()
    return redirect(url_for("shop"))

@app.route("/admin")
@login_required
def admin_panel():
    return render_template("admin.html")

@app.route("/api/accounts/all", methods=["GET"])
@login_required
def get_accounts_admin():
    db = load_db()
    return jsonify(db["accounts"])

@app.route("/api/upload", methods=["POST"])
@login_required
def upload_image():
    if "files" not in request.files:
        return jsonify({"success": False, "error": "Không có file"})
    files = request.files.getlist("files")
    urls = []
    for file in files:
        if file and file.filename and allowed_file(file.filename):
            ext = file.filename.rsplit(".", 1)[1].lower()
            filename = f"{secrets.token_hex(8)}.{ext}"
            filepath = os.path.join(app.config["UPLOAD_FOLDER"], filename)
            file.save(filepath)
            urls.append(f"/static/uploads/{filename}")
    if urls:
        return jsonify({"success": True, "urls": urls})
    return jsonify({"success": False, "error": "Không có file hợp lệ"})

@app.route("/api/accounts/add", methods=["POST"])
@login_required
def add_account():
    data = request.json
    db = load_db()
    acc_id = "ACC" + secrets.token_hex(4).upper()
    db["accounts"][acc_id] = {
        "id": acc_id,
        "name": data.get("name", "Acc FF"),
        "username": data.get("username", ""),
        "price": data.get("price", 0),
        "rank": data.get("rank", "Kim Cương"),
        "skin": data.get("skin", 0),
        "status": "available",
        "images": data.get("images", []),
        "created_at": time.time()
    }
    save_db(db)
    return jsonify({"success": True, "id": acc_id})

@app.route("/api/accounts/edit/<acc_id>", methods=["POST"])
@login_required
def edit_account(acc_id):
    data = request.json
    db = load_db()
    if acc_id not in db["accounts"]:
        return jsonify({"success": False, "error": "Acc không tồn tại"}), 404
    acc = db["accounts"][acc_id]
    for field in ["name", "username", "price", "rank", "skin", "images", "status"]:
        if field in data:
            acc[field] = data[field]
    acc["updated_at"] = time.time()
    save_db(db)
    return jsonify({"success": True})

@app.route("/api/accounts/delete/<acc_id>", methods=["DELETE"])
@login_required
def delete_account(acc_id):
    db = load_db()
    if acc_id in db["accounts"]:
        del db["accounts"][acc_id]
        save_db(db)
        return jsonify({"success": True})
    return jsonify({"success": False}), 404

# ========== SERVE UPLOAD ==========
@app.route("/static/uploads/<filename>")
def uploaded_file(filename):
    return send_from_directory(UPLOAD_FOLDER, filename)

if __name__ == "__main__":
    port = int(os.environ.get("PORT", 5000))
    print(f"[*] SHOPACC THEPHI chạy tại http://localhost:{port}")
    app.run(host="0.0.0.0", port=port, debug=True)