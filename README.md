# Ciphera
Learn. Build. Secure. Ciphera is an interactive cryptography platform that teaches the math, logic, and real-world use of modern encryption through hands-on tools and visual explanations.
from flask import Flask, request, jsonify
import hashlib

app = Flask(__name__)

@app.route("/hash", methods=["POST"])
def hash_text():
    data = request.json
    text = data.get("text", "")
    algo = data.get("algorithm", "sha256")

    if algo == "sha256":
        hashed = hashlib.sha256(text.encode()).hexdigest()
    elif algo == "sha1":
        hashed = hashlib.sha1(text.encode()).hexdigest()
    elif algo == "md5":
        hashed = hashlib.md5(text.encode()).hexdigest()
    else:
        return jsonify({"error": "Unsupported algorithm"}), 400

    return jsonify({
        "algorithm": algo,
        "hash": hashed
    })

@app.route("/caesar", methods=["POST"])
def caesar_cipher():
    data = request.json
    text = data.get("text", "")
    shift = int(data.get("shift", 3))

    result = ""
    for char in text:
        if char.isalpha():
            base = ord('a') if char.islower() else ord('A')
            result += chr((ord(char) - base + shift) % 26 + base)
        else:
            result += char

    return jsonify({"ciphertext": result})

if __name__ == "__main__":
    app.run(debug=True)
pip install flask
python app.py
<!DOCTYPE html>
<html>
<head>
  <title>Ciphera</title>
</head>
<body>
  <h1>🔐 Ciphera</h1>

  <h2>Hash Generator</h2>
  <input id="hashInput" placeholder="Enter text" />
  <select id="algo">
    <option value="sha256">SHA-256</option>
    <option value="sha1">SHA-1</option>
    <option value="md5">MD5</option>
  </select>
  <button onclick="hashText()">Hash</button>
  <pre id="hashOutput"></pre>

  <h2>Caesar Cipher</h2>
  <input id="caesarInput" placeholder="Message" />
  <input id="shift" type="number" value="3" />
  <button onclick="caesar()">Encrypt</button>
  <pre id="caesarOutput"></pre>

<script>
async function hashText() {
  const res = await fetch("/hash", {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({
      text: hashInput.value,
      algorithm: algo.value
    })
  });
  const data = await res.json();
  hashOutput.textContent = data.hash;
}

async function caesar() {
  const res = await fetch("/caesar", {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({
      text: caesarInput.value,
      shift: shift.value
    })
  });
  const data = await res.json();
  caesarOutput.textContent = data.ciphertext;
}
</script>
</body>
</html>
pip install cryptography
from cryptography.hazmat.primitives.asymmetric import rsa, padding
from cryptography.hazmat.primitives import serialization, hashes
from flask import jsonify, request
import base64

# Generate RSA key pair
def generate_rsa_keys():
    private_key = rsa.generate_private_key(
        public_exponent=65537,
        key_size=2048
    )

    public_key = private_key.public_key()

    priv_pem = private_key.private_bytes(
        encoding=serialization.Encoding.PEM,
        format=serialization.PrivateFormat.PKCS8,
        encryption_algorithm=serialization.NoEncryption()
    )

    pub_pem = public_key.public_bytes(
        encoding=serialization.Encoding.PEM,
        format=serialization.PublicFormat.SubjectPublicKeyInfo
    )

    return priv_pem.decode(), pub_pem.decode()

# Encrypt
def rsa_encrypt(public_key_pem, message):
    public_key = serialization.load_pem_public_key(
        public_key_pem.encode()
    )

    ciphertext = public_key.encrypt(
        message.encode(),
        padding.OAEP(
            mgf=padding.MGF1(algorithm=hashes.SHA256()),
            algorithm=hashes.SHA256(),
            label=None
        )
    )

    return base64.b64encode(ciphertext).decode()

# Decrypt
def rsa_decrypt(private_key_pem, ciphertext_b64):
    private_key = serialization.load_pem_private_key(
        private_key_pem.encode(),
        password=None
    )

    plaintext = private_key.decrypt(
        base64.b64decode(ciphertext_b64),
        padding.OAEP(
            mgf=padding.MGF1(algorithm=hashes.SHA256()),
            algorithm=hashes.SHA256(),
            label=None
        )
    )

    return plaintext.decode()
from rsa_routes import generate_rsa_keys, rsa_encrypt, rsa_decrypt

@app.route("/rsa/generate", methods=["GET"])
def rsa_generate():
    priv, pub = generate_rsa_keys()
    return jsonify({
        "private_key": priv,
        "public_key": pub
    })

@app.route("/rsa/encrypt", methods=["POST"])
def rsa_encrypt_route():
    data = request.json
    ciphertext = rsa_encrypt(
        data["public_key"],
        data["message"]
    )
    return jsonify({"ciphertext": ciphertext})

@app.route("/rsa/decrypt", methods=["POST"])
def rsa_decrypt_route():
    data = request.json
    plaintext = rsa_decrypt(
        data["private_key"],
        data["ciphertext"]
    )
    return jsonify({"plaintext": plaintext})
<h2>🔑 RSA Lab</h2>

<button onclick="generateKeys()">Generate Keys</button>

<h3>Public Key</h3>
<textarea id="pubKey" rows="6" cols="70"></textarea>

<h3>Private Key</h3>
<textarea id="privKey" rows="6" cols="70"></textarea>

<h3>Message</h3>
<input id="rsaMessage" placeholder="Secret message" />

<button onclick="encrypt()">Encrypt</button>
<pre id="cipherOut"></pre>

<button onclick="decrypt()">Decrypt</button>
<pre id="plainOut"></pre>

<script>
async function generateKeys() {
  const res = await fetch("/rsa/generate");
  const data = await res.json();
  pubKey.value = data.public_key;
  privKey.value = data.private_key;
}

async function encrypt() {
  const res = await fetch("/rsa/encrypt", {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({
      public_key: pubKey.value,
      message: rsaMessage.value
    })
  });
  const data = await res.json();
  cipherOut.textContent = data.ciphertext;
}

async function decrypt() {
  const res = await fetch("/rsa/decrypt", {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({
      private_key: privKey.value,
      ciphertext: cipherOut.textContent
    })
  });
  const data = await res.json();
  plainOut.textContent = data.plaintext;
}
</script>
pip install flask-login
from flask_login import UserMixin
import sqlite3

def get_db():
    return sqlite3.connect("ciphera.db")

class User(UserMixin):
    def __init__(self, id, email, password, is_pro):
        self.id = id
        self.email = email
        self.password = password
        self.is_pro = is_pro

def get_user_by_email(email):
    db = get_db()
    cur = db.cursor()
    cur.execute("SELECT * FROM users WHERE email = ?", (email,))
    row = cur.fetchone()
    return User(*row) if row else None

def get_user_by_id(user_id):
    db = get_db()
    cur = db.cursor()
    cur.execute("SELECT * FROM users WHERE id = ?", (user_id,))
    row = cur.fetchone()
    return User(*row) if row else None
import sqlite3

db = sqlite3.connect("ciphera.db")
db.execute("""
CREATE TABLE IF NOT EXISTS users (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  email TEXT UNIQUE,
  password TEXT,
  is_pro INTEGER DEFAULT 0
)
""")
db.commit()
from flask_login import LoginManager, login_user, login_required, logout_user, current_user
from werkzeug.security import generate_password_hash, check_password_hash
from models import get_user_by_email, get_user_by_id
login_manager = LoginManager()
login_manager.init_app(app)

@login_manager.user_loader
def load_user(user_id):
    return get_user_by_id(user_id)
@app.route("/register", methods=["POST"])
def register():
    data = request.json
    email = data["email"]
    password = generate_password_hash(data["password"])

    db = get_db()
    try:
        db.execute(
            "INSERT INTO users (email, password) VALUES (?, ?)",
            (email, password)
        )
        db.commit()
    except:
        return {"error": "User exists"}, 400

    return {"status": "registered"}
@app.route("/login", methods=["POST"])
def login():
    data = request.json
    user = get_user_by_email(data["email"])

    if not user or not check_password_hash(user.password, data["password"]):
        return {"error": "Invalid credentials"}, 401

    login_user(user)
    return {"status": "logged_in", "is_pro": user.is_pro}
@app.route("/logout")
@login_required
def logout():
    logout_user()
    return {"status": "logged_out"}
@app.route("/rsa/decrypt", methods=["POST"])
@login_required
def rsa_decrypt_route():
    if not current_user.is_pro:
        return {"error": "Pro feature"}, 403

    data = request.json
    plaintext = rsa_decrypt(
        data["private_key"],
        data["ciphertext"]
    )
    return {"plaintext": plaintext}
<h2>Account</h2>
<input id="email" placeholder="Email">
<input id="password" type="password" placeholder="Password">

<button onclick="register()">Register</button>
<button onclick="login()">Login</button>

<pre id="authStatus"></pre>

<script>
async function register() {
  const res = await fetch("/register", {
    method: "POST",
    headers: {"Content-Type":"application/json"},
    body: JSON.stringify({
      email: email.value,
      password: password.value
    })
  });
  authStatus.textContent = await res.text();
}

async function login() {
  const res = await fetch("/login", {
    method: "POST",
    headers: {"Content-Type":"application/json"},
    body: JSON.stringify({
      email: email.value,
      password: password.value
    })
  });
  authStatus.textContent = await res.text();
}
</script>
npm create vite@latest ciphera-ui -- --template react
cd ciphera-ui
npm install
npm run dev
src/
 ├─ components/
 │   ├─ Sidebar.jsx
 │   ├─ Dashboard.jsx
 │   ├─ RSALab.jsx
 │   └─ Login.jsx
 ├─ App.jsx
 └─ main.jsx
import { useState } from "react";
import Login from "./components/Login";
import Dashboard from "./components/Dashboard";

function App() {
  const [user, setUser] = useState(null);

  return (
    <>
      {user ? (
        <Dashboard user={user} setUser={setUser} />
      ) : (
        <Login setUser={setUser} />
      )}
    </>
  );
}

export default App;
function Login({ setUser }) {
  async function login(e) {
    e.preventDefault();

    const res = await fetch("http://localhost:5000/login", {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      credentials: "include",
      body: JSON.stringify({
        email: e.target.email.value,
        password: e.target.password.value
      })
    });

    const data = await res.json();
    if (res.ok) setUser(data);
  }

  return (
    <form onSubmit={login}>
      <h2>🔐 Ciphera Login</h2>
      <input name="email" placeholder="Email" />
      <input name="password" type="password" placeholder="Password" />
      <button>Login</button>
    </form>
  );
}

export default Login;
import Sidebar from "./Sidebar";
import RSALab from "./RSALab";

function Dashboard({ user, setUser }) {
  async function logout() {
    await fetch("http://localhost:5000/logout", {
      credentials: "include"
    });
    setUser(null);
  }

  return (
    <div style={{ display: "flex" }}>
      <Sidebar isPro={user.is_pro} logout={logout} />
      <main style={{ padding: "20px", width: "100%" }}>
        <h1>Welcome to Ciphera</h1>
        <RSALab isPro={user.is_pro} />
      </main>
    </div>
  );
}

export default Dashboard;
function Sidebar({ isPro, logout }) {
  return (
    <aside style={{ width: "220px", background: "#111", color: "#fff", padding: "20px" }}>
      <h2>Ciphera</h2>
      <p>Plan: {isPro ? "🔥 Pro" : "Free"}</p>

      <ul>
        <li>Dashboard</li>
        <li>Hashing</li>
        <li>RSA Lab</li>
        {isPro && <li>AES Lab</li>}
      </ul>

      <button onClick={logout}>Logout</button>
    </aside>
  );
}

export default Sidebar;
import { useState } from "react";

function RSALab({ isPro }) {
  const [pub, setPub] = useState("");
  const [priv, setPriv] = useState("");
  const [msg, setMsg] = useState("");
  const [out, setOut] = useState("");

  async function genKeys() {
    const res = await fetch("http://localhost:5000/rsa/generate");
    const data = await res.json();
    setPub(data.public_key);
    setPriv(data.private_key);
  }

  async function encrypt() {
    const res = await fetch("http://localhost:5000/rsa/encrypt", {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({ public_key: pub, message: msg })
    });
    const data = await res.json();
    setOut(data.ciphertext);
  }

  async function decrypt() {
    if (!isPro) {
      alert("🔒 Pro feature");
      return;
    }

    const res = await fetch("http://localhost:5000/rsa/decrypt", {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      credentials: "include",
      body: JSON.stringify({ private_key: priv, ciphertext: out })
    });
    const data = await res.json();
    setOut(data.plaintext);
  }

  return (
    <>
      <h2>🔑 RSA Lab</h2>
      <button onClick={genKeys}>Generate Keys</button>
      <textarea value={pub} readOnly />
      <textarea value={priv} readOnly />
      <input value={msg} onChange={e => setMsg(e.target.value)} />
      <button onClick={encrypt}>Encrypt</button>
      <button onClick={decrypt}>Decrypt (Pro)</button>
      <pre>{out}</pre>
    </>
  );
}

export default RSALab;
pip install cryptography
from cryptography.hazmat.primitives.ciphers.aead import AESGCM
import os, base64

def aes_encrypt(key_b64, plaintext):
    key = base64.b64decode(key_b64)
    aesgcm = AESGCM(key)

    nonce = os.urandom(12)  # 96-bit nonce (recommended)
    ciphertext = aesgcm.encrypt(nonce, plaintext.encode(), None)

    return {
        "nonce": base64.b64encode(nonce).decode(),
        "ciphertext": base64.b64encode(ciphertext).decode()
    }

def aes_decrypt(key_b64, nonce_b64, ciphertext_b64):
    key = base64.b64decode(key_b64)
    aesgcm = AESGCM(key)

    plaintext = aesgcm.decrypt(
        base64.b64decode(nonce_b64),
        base64.b64decode(ciphertext_b64),
        None
    )

    return plaintext.decode()

def generate_aes_key():
    return base64.b64encode(os.urandom(32)).decode()  # AES-256
from aes_routes import aes_encrypt, aes_decrypt, generate_aes_key
from flask_login import login_required, current_user

@app.route("/aes/key", methods=["GET"])
@login_required
def aes_key():
    if not current_user.is_pro:
        return {"error": "Pro feature"}, 403
    return {"key": generate_aes_key()}

@app.route("/aes/encrypt", methods=["POST"])
@login_required
def aes_encrypt_route():
    if not current_user.is_pro:
        return {"error": "Pro feature"}, 403

    data = request.json
    return aes_encrypt(data["key"], data["message"])

@app.route("/aes/decrypt", methods=["POST"])
@login_required
def aes_decrypt_route():
    if not current_user.is_pro:
        return {"error": "Pro feature"}, 403

    data = request.json
    plaintext = aes_decrypt(
        data["key"],
        data["nonce"],
        data["ciphertext"]
    )
    return {"plaintext": plaintext}
import { useState } from "react";

function AESLab({ isPro }) {
  const [key, setKey] = useState("");
  const [msg, setMsg] = useState("");
  const [nonce, setNonce] = useState("");
  const [cipher, setCipher] = useState("");
  const [out, setOut] = useState("");

  async function genKey() {
    if (!isPro) return alert("🔒 Pro feature");

    const res = await fetch("http://localhost:5000/aes/key", {
      credentials: "include"
    });
    const data = await res.json();
    setKey(data.key);
  }

  async function encrypt() {
    const res = await fetch("http://localhost:5000/aes/encrypt", {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      credentials: "include",
      body: JSON.stringify({ key, message: msg })
    });
    const data = await res.json();
    setNonce(data.nonce);
    setCipher(data.ciphertext);
  }

  async function decrypt() {
    const res = await fetch("http://localhost:5000/aes/decrypt", {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      credentials: "include",
      body: JSON.stringify({
        key,
        nonce,
        ciphertext: cipher
      })
    });
    const data = await res.json();
    setOut(data.plaintext);
  }

  return (
    <div>
      <h2>🔐 AES-256-GCM Lab</h2>

      <button onClick={genKey}>Generate AES Key (Pro)</button>
      <textarea value={key} readOnly placeholder="AES Key" />

      <input
        placeholder="Message"
        value={msg}
        onChange={e => setMsg(e.target.value)}
      />

      <button onClick={encrypt}>Encrypt</button>

      <textarea value={nonce} readOnly placeholder="Nonce" />
      <textarea value={cipher} readOnly placeholder="Ciphertext" />

      <button onClick={decrypt}>Decrypt</button>
      <pre>{out}</pre>
    </div>
  );
}

export default AESLab;
import AESLab from "./AESLab";

// inside <main>
{user.is_pro && <AESLab isPro={user.is_pro} />}
pip install stripe
import stripe
from flask import request, jsonify
from models import get_user_by_email, get_db

stripe.api_key = "sk_test_YOUR_SECRET_KEY"
PRICE_ID = "price_XXXXXXXX"
from flask_login import login_required, current_user

@login_required
def create_checkout():
    session = stripe.checkout.Session.create(
        customer_email=current_user.email,
        payment_method_types=["card"],
        mode="subscription",
        line_items=[{
            "price": PRICE_ID,
            "quantity": 1
        }],
        success_url="http://localhost:5173/success",
        cancel_url="http://localhost:5173/cancel"
    )
    return jsonify({ "url": session.url })
@app.route("/create-checkout", methods=["POST"])
@login_required
def checkout():
    return create_checkout()
@app.route("/stripe/webhook", methods=["POST"])
def stripe_webhook():
    payload = request.data
    sig = request.headers.get("Stripe-Signature")

    event = stripe.Webhook.construct_event(
        payload,
        sig,
        "whsec_YOUR_WEBHOOK_SECRET"
    )

    if event["type"] == "checkout.session.completed":
        session = event["data"]["object"]
        email = session["customer_email"]

        db = get_db()
        db.execute(
            "UPDATE users SET is_pro = 1 WHERE email = ?",
            (email,)
        )
        db.commit()

    return "", 200
POST /stripe/webhook
function Upgrade() {
  async function upgrade() {
    const res = await fetch("http://localhost:5000/create-checkout", {
      method: "POST",
      credentials: "include"
    });
    const data = await res.json();
    window.location.href = data.url;
  }

  return (
    <button onClick={upgrade}>
      🚀 Upgrade to Ciphera Pro
    </button>
  );
}

export default Upgrade;
{!user.is_pro && <Upgrade />}
function Success() {
  return (
    <div>
      <h1>🎉 Welcome to Ciphera Pro</h1>
      <p>Reload the app to unlock Pro features.</p>
    </div>
  );
}
from cryptography.hazmat.primitives.asymmetric import padding
from cryptography.hazmat.primitives import hashes, serialization
from cryptography.hazmat.primitives.ciphers.aead import AESGCM
import os, base64

def hybrid_encrypt(public_key_pem, message):
    # 1. Generate AES key
    aes_key = os.urandom(32)
    aesgcm = AESGCM(aes_key)
    nonce = os.urandom(12)

    # 2. Encrypt message with AES
    ciphertext = aesgcm.encrypt(nonce, message.encode(), None)

    # 3. Encrypt AES key with RSA public key
    public_key = serialization.load_pem_public_key(
        public_key_pem.encode()
    )

    encrypted_key = public_key.encrypt(
        aes_key,
        padding.OAEP(
            mgf=padding.MGF1(hashes.SHA256()),
            algorithm=hashes.SHA256(),
            label=None
        )
    )

    return {
        "encrypted_key": base64.b64encode(encrypted_key).decode(),
        "nonce": base64.b64encode(nonce).decode(),
        "ciphertext": base64.b64encode(ciphertext).decode()
    }

def hybrid_decrypt(private_key_pem, encrypted_key_b64, nonce_b64, ciphertext_b64):
    private_key = serialization.load_pem_private_key(
        private_key_pem.encode(),
        password=None
    )

    # 1. Decrypt AES key with RSA private key
    aes_key = private_key.decrypt(
        base64.b64decode(encrypted_key_b64),
        padding.OAEP(
            mgf=padding.MGF1(hashes.SHA256()),
            algorithm=hashes.SHA256(),
            label=None
        )
    )

    # 2. Decrypt message with AES
    aesgcm = AESGCM(aes_key)
    plaintext = aesgcm.decrypt(
        base64.b64decode(nonce_b64),
        base64.b64decode(ciphertext_b64),
        None
    )

    return plaintext.decode()
from hybrid_routes import hybrid_encrypt, hybrid_decrypt
from flask_login import login_required, current_user

@app.route("/hybrid/encrypt", methods=["POST"])
@login_required
def hybrid_encrypt_route():
    if not current_user.is_pro:
        return {"error": "Pro feature"}, 403

    data = request.json
    return hybrid_encrypt(
        data["public_key"],
        data["message"]
    )

@app.route("/hybrid/decrypt", methods=["POST"])
@login_required
def hybrid_decrypt_route():
    if not current_user.is_pro:
        return {"error": "Pro feature"}, 403

    data = request.json
    plaintext = hybrid_decrypt(
        data["private_key"],
        data["encrypted_key"],
        data["nonce"],
        data["ciphertext"]
    )
    return {"plaintext": plaintext}
import { useState } from "react";

function HybridLab({ isPro }) {
  const [pub, setPub] = useState("");
  const [priv, setPriv] = useState("");
  const [msg, setMsg] = useState("");
  const [ek, setEk] = useState("");
  const [nonce, setNonce] = useState("");
  const [cipher, setCipher] = useState("");
  const [out, setOut] = useState("");

  async function encrypt() {
    if (!isPro) return alert("🔒 Pro feature");

    const res = await fetch("http://localhost:5000/hybrid/encrypt", {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      credentials: "include",
      body: JSON.stringify({ public_key: pub, message: msg })
    });
    const data = await res.json();
    setEk(data.encrypted_key);
    setNonce(data.nonce);
    setCipher(data.ciphertext);
  }

  async function decrypt() {
    const res = await fetch("http://localhost:5000/hybrid/decrypt", {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      credentials: "include",
      body: JSON.stringify({
        private_key: priv,
        encrypted_key: ek,
        nonce,
        ciphertext: cipher
      })
    });
    const data = await res.json();
    setOut(data.plaintext);
  }

  return (
    <div>
      <h2>🔗 Hybrid Encryption (RSA + AES)</h2>

      <textarea placeholder="RSA Public Key" onChange={e => setPub(e.target.value)} />
      <textarea placeholder="RSA Private Key" onChange={e => setPriv(e.target.value)} />

      <input
        placeholder="Message"
        value={msg}
        onChange={e => setMsg(e.target.value)}
      />

      <button onClick={encrypt}>Encrypt (Hybrid)</button>

      <textarea value={ek} readOnly placeholder="Encrypted AES Key" />
      <textarea value={nonce} readOnly placeholder="Nonce" />
      <textarea value={cipher} readOnly placeholder="Ciphertext" />

      <button onClick={decrypt}>Decrypt</button>
      <pre>{out}</pre>
    </div>
  );
}

export default HybridLab;
import HybridLab from "./HybridLab";

{user.is_pro && <HybridLab isPro={user.is_pro} />}
React (Vercel)  →  Flask API (Render)  →  SQLite (later Postgres)
                     │
                     └─ Stripe Webhooks
Flask
flask-login
cryptography
stripe
gunicorn
app.config.update(
    SESSION_COOKIE_SAMESITE="None",
    SESSION_COOKIE_SECURE=True
)
pip install flask-cors
from flask_cors import CORS
CORS(app, supports_credentials=True)
git init
git add .
git commit -m "Deploy Ciphera backend"
git remote add origin https://github.com/YOURNAME/ciphera-backend.git
git push -u origin main
pip install -r requirements.txt
gunicorn app:app
STRIPE_SECRET_KEY=sk_live_...
STRIPE_WEBHOOK_SECRET=whsec_...
PRICE_ID=price_...
FLASK_ENV=production
import os
stripe.api_key = os.getenv("STRIPE_SECRET_KEY")
https://ciphera-api.onrender.com/stripe/webhook
http://localhost:5000
https://ciphera-api.onrender.com
credentials: "include"
npm install -g vercel
vercel
login_user(user)
response = jsonify({"status": "logged_in", "is_pro": user.is_pro})
response.headers.add(
    "Access-Control-Allow-Credentials", "true"
)
return response
CORS(app, supports_credentials=True)
db.execute("""
CREATE TABLE IF NOT EXISTS progress (
  user_id INTEGER,
  lab TEXT,
  completed INTEGER DEFAULT 0,
  completed_at TEXT,
  PRIMARY KEY (user_id, lab)
)
""")
db.commit()
import datetime
from models import get_db

def mark_completed(user_id, lab):
    db = get_db()
    db.execute("""
        INSERT INTO progress (user_id, lab, completed, completed_at)
        VALUES (?, ?, 1, ?)
        ON CONFLICT(user_id, lab)
        DO UPDATE SET completed = 1, completed_at = ?
    """, (user_id, lab, datetime.datetime.utcnow(), datetime.datetime.utcnow()))
    db.commit()

def get_progress(user_id):
    db = get_db()
    cur = db.cursor()
    cur.execute("""
        SELECT lab, completed FROM progress
        WHERE user_id = ?
    """, (user_id,))
    return {lab: completed for lab, completed in cur.fetchall()}
from flask_login import login_required, current_user
from progress import mark_completed, get_progress

@app.route("/progress/complete", methods=["POST"])
@login_required
def complete_lab():
    lab = request.json["lab"]
    mark_completed(current_user.id, lab)
    return {"status": "ok"}
@app.route("/progress", methods=["GET"])
@login_required
def progress():
    return get_progress(current_user.id)
mark_completed(current_user.id, "hybrid")
import { useEffect, useState } from "react";

const LABS = ["hashing", "rsa", "aes", "hybrid"];

function Progress() {
  const [progress, setProgress] = useState({});

  useEffect(() => {
    fetch("https://ciphera-api.onrender.com/progress", {
      credentials: "include"
    })
      .then(res => res.json())
      .then(setProgress);
  }, []);

  const completed = LABS.filter(l => progress[l]).length;
  const percent = Math.round((completed / LABS.length) * 100);

  return (
    <div>
      <h2>📊 Your Progress</h2>
      <p>{percent}% complete</p>

      <ul>
        {LABS.map(lab => (
          <li key={lab}>
            {progress[lab] ? "✅" : "⬜"} {lab.toUpperCase()}
          </li>
        ))}
      </ul>
    </div>
  );
}

export default Progress;
import Progress from "./Progress";

<Progress />
challenges (
  id INTEGER PRIMARY KEY,
  title TEXT,
  description TEXT,
  difficulty TEXT, -- beginner | intermediate | advanced
  category TEXT,   -- AES, RSA, Hybrid
  is_pro INTEGER
)

user_challenges (
  user_id INTEGER,
  challenge_id INTEGER,
  completed INTEGER,
  attempts INTEGER,
  completed_at DATETIME
)
@app.route("/api/challenges")
@login_required
def get_challenges():
    db = get_db()
    rows = db.execute("""
        SELECT c.*, 
        COALESCE(uc.completed, 0) as completed
        FROM challenges c
        LEFT JOIN user_challenges uc
        ON c.id = uc.challenge_id AND uc.user_id = ?
    """, (current_user.id,)).fetchall()
    return jsonify([dict(r) for r in rows])
@app.route("/api/challenges/submit", methods=["POST"])
@login_required
def submit_challenge():
    data = request.json
    challenge_id = data["challenge_id"]
    correct = verify_solution(challenge_id, data)

    db = get_db()
    if correct:
        db.execute("""
            INSERT OR REPLACE INTO user_challenges
            (user_id, challenge_id, completed, attempts)
            VALUES (?, ?, 1, 1)
        """, (current_user.id, challenge_id))
    else:
        db.execute("""
            UPDATE user_challenges
            SET attempts = attempts + 1
            WHERE user_id = ? AND challenge_id = ?
        """, (current_user.id, challenge_id))

    db.commit()
    return jsonify({"correct": correct})
📚 Challenges
━━━━━━━━━━━━━━━━━━━━
🔓 AES Basics        ✅ Completed
🔐 RSA Keys          ⏳ In Progress
🧠 Hybrid Encryption 🔒 Pro
import os
from cryptography.hazmat.primitives.ciphers.aead import AESGCM
import base64

def generate_aes_challenge():
    key = AESGCM.generate_key(bit_length=256)
    iv = os.urandom(12)
    plaintext = b"ciphera_is_secure"

    aesgcm = AESGCM(key)
    ciphertext = aesgcm.encrypt(iv, plaintext, None)

    return {
        "plaintext": plaintext.decode(),
        "key": base64.b64encode(key).decode(),
        "iv": base64.b64encode(iv).decode(),
        "expected": base64.b64encode(ciphertext).decode()
    }
def grade_aes_submission(user_ciphertext_b64, expected_b64):
    try:
        return user_ciphertext_b64 == expected_b64
    except Exception:
        return False
from cryptography.hazmat.primitives.asymmetric import rsa, padding
from cryptography.hazmat.primitives import serialization, hashes

def generate_rsa_challenge():
    private_key = rsa.generate_private_key(
        public_exponent=65537,
        key_size=2048
    )
    public_key = private_key.public_key()

    message = b"aes_session_key"

    encrypted = public_key.encrypt(
        message,
        padding.OAEP(
            mgf=padding.MGF1(hashes.SHA256()),
            algorithm=hashes.SHA256(),
            label=None
        )
    )

    return {
        "public_key": public_key.public_bytes(
            serialization.Encoding.PEM,
            serialization.PublicFormat.SubjectPublicKeyInfo
        ).decode(),
        "expected": base64.b64encode(encrypted).decode(),
        "private_key": private_key  # server-only
    }
def grade_rsa_submission(user_ciphertext_b64, private_key):
    try:
        decrypted = private_key.decrypt(
            base64.b64decode(user_ciphertext_b64),
            padding.OAEP(
                mgf=padding.MGF1(hashes.SHA256()),
                algorithm=hashes.SHA256(),
                label=None
            )
        )
        return decrypted == b"aes_session_key"
    except Exception:
        return False
{
  "aes_ciphertext": "...",
  "iv": "...",
  "rsa_encrypted_key": "..."
}
def grade_hybrid_submission(data, private_key):
    try:
        # 1. Decrypt AES key using RSA
        aes_key = private_key.decrypt(
            base64.b64decode(data["rsa_encrypted_key"]),
            padding.OAEP(
                mgf=padding.MGF1(hashes.SHA256()),
                algorithm=hashes.SHA256(),
                label=None
            )
        )

        # 2. Decrypt message using AES
        aesgcm = AESGCM(aes_key)
        plaintext = aesgcm.decrypt(
            base64.b64decode(data["iv"]),
            base64.b64decode(data["aes_ciphertext"]),
            None
        )

        return plaintext == b"ciphera_hybrid_success"
    except Exception:
        return False
ALTER TABLE user_challenges ADD COLUMN started_at DATETIME;
ALTER TABLE user_challenges ADD COLUMN suspicious INTEGER DEFAULT 0;
@app.route("/api/challenges/start", methods=["POST"])
@login_required
def start_challenge():
    challenge_id = request.json["challenge_id"]
    db = get_db()
    db.execute("""
        INSERT OR REPLACE INTO user_challenges
        (user_id, challenge_id, started_at, attempts)
        VALUES (?, ?, CURRENT_TIMESTAMP, 0)
    """, (current_user.id, challenge_id))
    db.commit()
    return "", 200
def is_too_fast(started_at, min_seconds):
    elapsed = (datetime.utcnow() - started_at).total_seconds()
    return elapsed < min_seconds
import hashlib

def hash_submission(data):
    raw = json.dumps(data, sort_keys=True).encode()
    return hashlib.sha256(raw).hexdigest()
CREATE TABLE submissions (
  user_id INTEGER,
  challenge_id INTEGER,
  solution_hash TEXT,
  created_at DATETIME
);
def is_reused_solution(challenge_id, solution_hash, db):
    row = db.execute("""
        SELECT COUNT(*) as count
        FROM submissions
        WHERE challenge_id = ?
        AND solution_hash = ?
    """, (challenge_id, solution_hash)).fetchone()
    return row["count"] > 0
def validate_hybrid_structure(data):
    required = {"aes_ciphertext", "iv", "rsa_encrypted_key"}
    return required.issubset(data.keys())
def is_spam_attempt(attempts, started_at):
    elapsed = (datetime.utcnow() - started_at).total_seconds()
    return attempts >= 5 and elapsed < 30
def is_spam_attempt(attempts, started_at):
    elapsed = (datetime.utcnow() - started_at).total_seconds()
    return attempts >= 5 and elapsed < 30
def anti_cheat_check(ctx):
    flags = []

    if ctx["too_fast"]:
        flags.append("too_fast")

    if ctx["reused"]:
        flags.append("solution_reuse")

    if ctx["spam"]:
        flags.append("spam_attempts")

    if ctx["invalid_structure"]:
        flags.append("step_skipping")

    return flags
if flags:
    db.execute("""
        UPDATE user_challenges
        SET suspicious = 1
        WHERE user_id = ? AND challenge_id = ?
    """, (user_id, challenge_id))
from functools import wraps
from flask_login import current_user
from flask import abort

def pro_required(f):
    @wraps(f)
    def wrapper(*args, **kwargs):
        if not current_user.is_pro:
            abort(403, description="Pro subscription required")
        return f(*args, **kwargs)
    return wrapper
@app.route("/api/challenges/advanced")
@login_required
@pro_required
def advanced_challenges():
    ...
def compute_user_level(completed_count):
    if completed_count < 5:
        return "Novice"
    elif completed_count < 15:
        return "Practitioner"
    elif completed_count < 30:
        return "Engineer"
    else:
        return "Cipher Master"
@app.route("/api/user/level")
@login_required
def user_level():
    db = get_db()
    count = db.execute("""
        SELECT COUNT(*) as c
        FROM user_challenges
        WHERE user_id = ? AND completed = 1
    """, (current_user.id,)).fetchone()["c"]

    return jsonify({
        "level": compute_user_level(count),
        "completed": count
    })
@app.route("/api/leaderboard")
def leaderboard():
    db = get_db()
    rows = db.execute("""
        SELECT u.username, COUNT(uc.challenge_id) as score
        FROM users u
        JOIN user_challenges uc ON u.id = uc.user_id
        WHERE uc.completed = 1 AND uc.suspicious = 0
        GROUP BY u.id
        ORDER BY score DESC
        LIMIT 10
    """).fetchall()

    return jsonify([dict(r) for r in rows])
@app.route("/api/challenges/reset", methods=["POST"])
@login_required
def reset_challenge():
    challenge_id = request.json["challenge_id"]

    db = get_db()
    db.execute("""
        DELETE FROM user_challenges
        WHERE user_id = ? AND challenge_id = ?
    """, (current_user.id, challenge_id))
    db.commit()

    return jsonify({"status": "reset"})
CREATE TABLE challenge_instances (
  id INTEGER PRIMARY KEY,
  user_id INTEGER,
  challenge_id INTEGER,
  secret_data TEXT,
  created_at DATETIME
);
def save_challenge_instance(user_id, challenge_id, secret_data):
    db = get_db()
    db.execute("""
        INSERT INTO challenge_instances
        (user_id, challenge_id, secret_data)
        VALUES (?, ?, ?)
    """, (user_id, challenge_id, json.dumps(secret_data)))
    db.commit()
CREATE TABLE hints (
  challenge_id INTEGER,
  step INTEGER,
  text TEXT
);
@app.route("/api/challenges/hint")
@login_required
@pro_required
def get_hint():
    challenge_id = request.args["challenge_id"]
    step = request.args.get("step", 1)

    db = get_db()
    hint = db.execute("""
        SELECT text FROM hints
        WHERE challenge_id = ? AND step = ?
    """, (challenge_id, step)).fetchone()

    return jsonify({"hint": hint["text"]})
CREATE TABLE audit_log (
  user_id INTEGER,
  challenge_id INTEGER,
  action TEXT,
  metadata TEXT,
  created_at DATETIME
);
def log_event(user_id, challenge_id, action, metadata=None):
    db = get_db()
    db.execute("""
        INSERT INTO audit_log
        VALUES (?, ?, ?, ?, CURRENT_TIMESTAMP)
    """, (user_id, challenge_id, action, json.dumps(metadata or {})))
    db.commit()
import { createContext, useContext } from "react";

export const AuthContext = createContext(null);

export function useAuth() {
  return useContext(AuthContext);
}
<AuthContext.Provider value={{ user, isPro }}>
  <App />
</AuthContext.Provider>
function ChallengeCard({ challenge }) {
  if (challenge.is_pro && !isPro) {
    return <div className="locked">🔒 Pro Only</div>;
  }

  return <div>{challenge.title}</div>;
}
/admin
/admin/users
/admin/challenges
/admin/submissions
/admin/revenue
/admin/securityALTER TABLE users ADD COLUMN role TEXT DEFAULT 'user';
user
admin
super_adminfrom flask import abort
from flask_login import current_user
from functools import wraps

def admin_required(f):
    @wraps(f)
    def wrapper(*args, **kwargs):
        if current_user.role not in ["admin", "super_admin"]:
            abort(403)
        return f(*args, **kwargs)
    return wrapper@app.route("/admin/stats")
@login_required
@admin_required
def admin_stats():
    db = get_db()

    users = db.execute("SELECT COUNT(*) as c FROM users").fetchone()["c"]
    pro_users = db.execute("SELECT COUNT(*) as c FROM users WHERE is_pro = 1").fetchone()["c"]
    submissions = db.execute("SELECT COUNT(*) as c FROM submissions").fetchone()["c"]
    suspicious = db.execute("SELECT COUNT(*) as c FROM user_challenges WHERE suspicious = 1").fetchone()["c"]

    return jsonify({
        "users": users,
        "pro_users": pro_users,
        "submissions": submissions,
        "suspicious_flags": suspicious
    })@app.route("/admin/users")
@login_required
@admin_required
def admin_users():
    db = get_db()

    rows = db.execute("""
        SELECT id, email, is_pro, role
        FROM users
        ORDER BY id DESC
    """).fetchall()

    return jsonify([dict(r) for r in rows])@app.route("/admin/users")
@login_required
@admin_required
def admin_users():
    db = get_db()

    rows = db.execute("""
        SELECT id, email, is_pro, role
        FROM users
        ORDER BY id DESC
    """).fetchall()

    return jsonify([dict(r) for r in rows])
    @app.route("/admin/users/ban", methods=["POST"])
@login_required
@admin_required
def ban_user():
    user_id = request.json["user_id"]

    db = get_db()
    db.execute(
        "UPDATE users SET banned = 1 WHERE id = ?",
        (user_id,)
    )
    db.commit()

    return jsonify({"status": "banned"})@app.route("/admin/stats")
@login_required
@admin_required
def admin_stats():
    db = get_db()

    users = db.execute("SELECT COUNT(*) as c FROM users").fetchone()["c"]
    pro_users = db.execute("SELECT COUNT(*) as c FROM users WHERE is_pro = 1").fetchone()["c"]
    submissions = db.execute("SELECT COUNT(*) as c FROM submissions").fetchone()["c"]
    suspicious = db.execute("SELECT COUNT(*) as c FROM user_challenges WHERE suspicious = 1").fetchone()["c"]

    return jsonify({
        "users": users,
        "pro_users": pro_users,
        "submissions": submissions,
        "suspicious_flags": suspicious
    })@app.route("/admin/users")
@login_required
@admin_required
def admin_users():
    db = get_db()

    rows = db.execute("""
        SELECT id, email, is_pro, role
        FROM users
        ORDER BY id DESC
    """).fetchall()

    return jsonify([dict(r) for r in rows])
    ALTER TABLE users ADD COLUMN banned INTEGER DEFAULT 0;@app.route("/admin/suspicious")
@login_required
@admin_required
def suspicious_activity():
    db = get_db()

    rows = db.execute("""
        SELECT user_id, challenge_id, attempts, suspicious
        FROM user_challenges
        WHERE suspicious = 1
    """).fetchall()

    return jsonify([dict(r) for r in rows])@app.route("/admin/revenue")
@login_required
@admin_required
def revenue():
    db = get_db()

    pro = db.execute(
        "SELECT COUNT(*) as c FROM users WHERE is_pro = 1"
    ).fetchone()["c"]

    monthly = pro * 10

    return jsonify({
        "pro_users": pro,
        "estimated_mrr": monthly
    })@app.route("/admin/revenue")
@login_required
@admin_required
def revenue():
    db = get_db()

    pro = db.execute(
        "SELECT COUNT(*) as c FROM users WHERE is_pro = 1"
    ).fetchone()["c"]

    monthly = pro * 10

    return jsonify({
        "pro_users": pro,
        "estimated_mrr": monthly
    })@app.route("/admin/revenue")
@login_required
@admin_required
def revenue():
    db = get_db()

    pro = db.execute(
        "SELECT COUNT(*) as c FROM users WHERE is_pro = 1"
    ).fetchone()["c"]

    monthly = pro * 10

    return jsonify({
        "pro_users": pro,
        "estimated_mrr": monthly
    })function AdminDashboard() {
  const [stats, setStats] = useState(null)

  useEffect(() => {
    fetch("/admin/stats")
      .then(r => r.json())
      .then(setStats)
  }, [])

  if (!stats) return <div>Loading...</div>

  return (
    <div>
      <h1>Admin Dashboard</h1>

      <div>Users: {stats.users}</div>
      <div>Pro Users: {stats.pro_users}</div>
      <div>Submissions: {stats.submissions}</div>
      <div>Suspicious Flags: {stats.suspicious_flags}</div>
    </div>
  )
}pip install bcryptimport bcrypt

def hash_password(password):
    return bcrypt.hashpw(password.encode(), bcrypt.gensalt())

def check_password(password, hashed):
    return bcrypt.checkpw(password.encode(), hashed)pip install flask-wtffrom flask_wtf.csrf import CSRFProtect

csrf = CSRFProtect(app)@app.after_request
def set_security_headers(resp):

    resp.headers["X-Frame-Options"] = "DENY"
    resp.headers["X-Content-Type-Options"] = "nosniff"
    resp.headers["X-XSS-Protection"] = "1; mode=block"
    resp.headers["Content-Security-Policy"] = "default-src 'self'"

    return respfrom flask_limiter import Limiter

limiter = Limiter(app)

@app.route("/login", methods=["POST"])
@limiter.limit("5 per minute")
def login():
    ...def log_admin_login(user_id):

    db = get_db()

    db.execute("""
        INSERT INTO audit_log
        (user_id, action, created_at)
        VALUES (?, 'admin_login', CURRENT_TIMESTAMP)
    """, (user_id,))

    db.commit()"whsec_YOUR_WEBHOOK_SECRET"import os

STRIPE_WEBHOOK_SECRET = os.getenv("STRIPE_WEBHOOK_SECRET")/admin/*if not isinstance(data["aes_ciphertext"], str):
    abort(400)
