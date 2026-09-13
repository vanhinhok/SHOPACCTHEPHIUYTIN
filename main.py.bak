#!/usr/bin/env python3
# -*- coding: utf-8 -*-
# SHOPACC THEPHI - Gộp shop + admin, PostgreSQL + Cloudinary

from flask import Flask, render_template, request, jsonify, session, redirect, url_for, send_from_directory
import os, secrets, time, hashlib
from functools import wraps
from werkzeug.utils import secure_filename
import psycopg2
from psycopg2.extras import RealDictCursor
import cloudinary
import cloudinary.uploader

app = Flask(__name__, template_folder="templates", static_folder="static", static_url_path="/static")
app.secret_key = secrets.token_hex(32)

DATABASE_URL = os.environ.get("DATABASE_URL")
UPLOAD_FOLDER = "static/uploads"
ALLOWED_EXTENSIONS = {"png", "jpg", "jpeg", "gif", "webp"}
os.makedirs(UPLOAD_FOLDER, exist_ok=True)
app.config["UPLOAD_FOLDER"] = UPLOAD_FOLDER
app.config["MAX_CONTENT_LENGTH"] = 20 * 1024 * 1024

# ========== CLOUDINARY ==========
cloudinary.config(
    cloud_name=os.environ.get("CLOUDINARY_CLOUD_NAME"),
    api_key=os.environ.get("CLOUDINARY_API_KEY"),
    api_secret=os.environ.get("CLOUDINARY_API_SECRET")
)

# ========== DATABASE ==========
def get_conn():
    return psycopg2.connect(DATABASE_URL, sslmode="require")

def init_db():
    conn = get_conn()
    cur = conn.cursor()
    cur.execute("""
        CREATE TABLE IF NOT EXISTS accounts (
            id VARCHAR(20) PRIMARY KEY,
            name TEXT, username TEXT, price INT,
            rank TEXT, skin INT, status TEXT,
            images TEXT[], created_at FLOAT
        )
    """)
    cur.execute("""
        CREATE TABLE IF NOT EXISTS admins (
            username TEXT PRIMARY KEY, password TEXT
        )
    """)
    cur.execute("SELECT COUNT(*) FROM admins")
    if cur.fetchone()[0] == 0:
        cur.execute("INSERT INTO admins (username, password) VALUES (%s, %s)",
                    ("admin", hashlib.sha256("admin123".encode()).hexdigest()))
    conn.commit()
    cur.close(); conn.close()

init_db()

def allowed_file(filename):
    return "." in filename and filename.rsplit(".", 1)[1].lower() in ALLOWED_EXTENSIONS

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
    conn = get_conn()
    cur = conn.cursor(cursor_factory=RealDictCursor)
    cur.execute("SELECT id, name, price, rank, skin, status, images FROM accounts")
    rows = cur.fetchall()
    cur.close(); conn.close()
    return jsonify({r["id"]: dict(r) for r in rows})

# ========== ADMIN ==========
@app.route("/login", methods=["GET", "POST"])
def login():
    if request.method == "POST":
        u = request.form.get("username")
        p = request.form.get("password")
        h = hashlib.sha256(p.encode()).hexdigest()
        conn = get_conn()
        cur = conn.cursor()
        cur.execute("SELECT password FROM admins WHERE username=%s", (u,))
        row = cur.fetchone()
        cur.close(); conn.close()
        if row and row[0] == h:
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
    conn = get_conn()
    cur = conn.cursor(cursor_factory=RealDictCursor)
    cur.execute("SELECT * FROM accounts ORDER BY created_at DESC")
    rows = cur.fetchall()
    cur.close(); conn.close()
    return jsonify({r["id"]: dict(r) for r in rows})

# ========== UPLOAD (CLOUDINARY) ==========
@app.route("/api/upload", methods=["POST"])
@login_required
def upload_image():
    if "files" not in request.files:
        return jsonify({"success": False, "error": "Không có file"})
    files = request.files.getlist("files")
    urls = []
    for file in files:
        if file and file.filename and allowed_file(file.filename):
            try:
                result = cloudinary.uploader.upload(file)
                urls.append(result["secure_url"])
            except Exception as e:
                return jsonify({"success": False, "error": str(e)})
    if urls:
        return jsonify({"success": True, "urls": urls})
    return jsonify({"success": False, "error": "Không có file hợp lệ"})

@app.route("/api/accounts/add", methods=["POST"])
@login_required
def add_account():
    data = request.json
    acc_id = "ACC" + secrets.token_hex(4).upper()
    conn = get_conn()
    cur = conn.cursor()
    cur.execute("""
        INSERT INTO accounts (id, name, username, price, rank, skin, status, images, created_at)
        VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s)
    """, (
        acc_id,
        data.get("name", "Acc FF"),
        data.get("username", ""),
        data.get("price", 0),
        data.get("rank", "Kim Cương"),
        data.get("skin", 0),
        "available",
        data.get("images", []),
        time.time()
    ))
    conn.commit()
    cur.close(); conn.close()
    return jsonify({"success": True, "id": acc_id})

@app.route("/api/accounts/edit/<acc_id>", methods=["POST"])
@login_required
def edit_account(acc_id):
    data = request.json
    conn = get_conn()
    cur = conn.cursor()
    fields = []
    values = []
    for f in ["name", "username", "price", "rank", "skin", "images", "status"]:
        if f in data:
            fields.append(f"{f}=%s")
            values.append(data[f])
    if not fields:
        cur.close(); conn.close()
        return jsonify({"success": False, "error": "Không có dữ liệu"})
    values.append(acc_id)
    cur.execute(f"UPDATE accounts SET {','.join(fields)} WHERE id=%s", values)
    conn.commit()
    cur.close(); conn.close()
    return jsonify({"success": True})

@app.route("/api/accounts/delete/<acc_id>", methods=["DELETE"])
@login_required
def delete_account(acc_id):
    conn = get_conn()
    cur = conn.cursor()
    cur.execute("DELETE FROM accounts WHERE id=%s", (acc_id,))
    conn.commit()
    cur.close(); conn.close()
    return jsonify({"success": True})

# ========== SERVE UPLOAD (fallback) ==========
@app.route("/static/uploads/<filename>")
def uploaded_file(filename):
    return send_from_directory(UPLOAD_FOLDER, filename)

if __name__ == "__main__":
    port = int(os.environ.get("PORT", 5000))
    print(f"[*] SHOPACC THEPHI chạy tại http://localhost:{port}")
    app.run(host="0.0.0.0", port=port, debug=True)