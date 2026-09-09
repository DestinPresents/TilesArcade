/* Tile Trails Firebase Auth + Firestore cloud save */
(function () {
  "use strict";

  var auth = null;
  var db = null;
  var currentUser = null;
  var profile = { name: "", playerId: "" };
  var saveTimer = null;
  var ready = false;

  /* Hard fallback prevents "Firebase configuration not defined" even if the
     config script is accidentally loaded after this file. */
  var FALLBACK_CONFIG = {
    apiKey: "AIzaSyBMaB2v9lbM9r2M4Nv31hRT6vImaenOQEc",
    authDomain: "tiles-96c54.firebaseapp.com",
    databaseURL: "https://tiles-96c54-default-rtdb.asia-southeast1.firebasedatabase.app",
    projectId: "tiles-96c54",
    storageBucket: "tiles-96c54.firebasestorage.app",
    messagingSenderId: "511133312503",
    appId: "1:511133312503:web:7c8735404949fef93f3af2"
  };

  function cfg() {
    return (window.firebaseConfig && typeof window.firebaseConfig === "object")
      ? window.firebaseConfig
      : FALLBACK_CONFIG;
  }

  function $(id) { return document.getElementById(id); }
  function safeName(v) { return String(v || "").trim().replace(/\s+/g, " ").slice(0, 24); }
  function fallbackName(u) { return safeName(u && u.displayName) || safeName(u && u.email ? u.email.split("@")[0] : "Player") || "Player"; }
  function makePlayerId() { return "TT-" + Math.floor(10000000 + Math.random() * 90000000); }
  function escapeHtml(v) { return String(v || "").replace(/[&<>"']/g, function (m) { return ({"&":"&amp;","<":"&lt;",">":"&gt;","\"":"&quot;","'":"&#39;"})[m]; }); }

  function setStatus(msg, error) {
    var el = $("authStatus");
    if (!el) return;
    el.textContent = msg || "";
    el.className = error ? "auth-error" : "auth-status";
    el.style.display = msg ? "block" : "none";
  }

  function setAccountView() {
    var label = $("accountLabel"), id = $("accountPlayerId");
    if (!currentUser) {
      if (label) label.textContent = "LOGIN";
      if (id) id.textContent = "";
      return;
    }
    if (label) label.textContent = profile.name || fallbackName(currentUser);
    if (id) id.textContent = profile.playerId || "";
  }

  function renderAuthForm(mode, message) {
    var body = $("accountBody");
    if (!body) return;
    message = message || "";
    body.innerHTML = `
      <h2>${mode === "login" ? "Player Login" : "Create Account"}</h2>
      <div class="auth-tabs">
        <button class="${mode === "login" ? "active" : ""}" onclick="window.TileTrailsAuth.render('login')">LOGIN</button>
        <button class="${mode === "signup" ? "active" : ""}" onclick="window.TileTrailsAuth.render('signup')">SIGN UP</button>
      </div>
      <div id="authStatus" class="${message ? "auth-error" : "auth-status"}" style="${message ? "" : "display:none"}">${escapeHtml(message)}</div>
      <form class="auth-form" id="authForm">
        ${mode === "signup" ? '<label>Player Name<input id="signupName" maxlength="24" autocomplete="name" placeholder="Your game name" required></label>' : ""}
        <label>Email<input id="authEmail" type="email" autocomplete="email" placeholder="you@example.com" required></label>
        <label>Password
          <span style="display:flex;gap:8px;align-items:center">
            <input id="authPassword" type="password" minlength="6" autocomplete="${mode === "login" ? "current-password" : "new-password"}" placeholder="Minimum 6 characters" required style="flex:1">
            <button type="button" id="togglePassword" aria-label="Show password" style="min-width:46px;padding:11px 10px;border-radius:12px;background:#fff;border:2px solid #9dd9ef;color:#075e8e;font-weight:1000;cursor:pointer">👁️</button>
          </span>
        </label>
        <button class="modal-action" type="submit">${mode === "login" ? "LOGIN" : "CREATE ACCOUNT"}</button>
      </form>
      <button class="secondary-action full" onclick="window.TileTrailsAuth.guestHint()">Continue as guest</button>`;
    var toggle = $("togglePassword");
    if (toggle) toggle.onclick = function () {
      var input = $("authPassword");
      if (!input) return;
      var visible = input.type === "text";
      input.type = visible ? "password" : "text";
      toggle.textContent = visible ? "👁️" : "🙈";
      toggle.setAttribute("aria-label", visible ? "Show password" : "Hide password");
    };
    $("authForm").onsubmit = function (e) {
      e.preventDefault();
      mode === "login" ? login() : signup();
    };
  }

  function openAccount() {
    var body = $("accountBody");
    if (!body) return;
    if (currentUser) {
      body.innerHTML = `
        <div class="account-welcome">
          <span class="account-badge">PLAYER ACCOUNT</span>
          <h2>Welcome, <strong>${escapeHtml(profile.name || fallbackName(currentUser))}</strong></h2>
          <small class="player-id-line">PLAYER ID: ${escapeHtml(profile.playerId)}</small>
        </div>
        <div class="account-menu">
          <button onclick="window.TileTrailsAuth.syncNow()">☁️ SAVE TO CLOUD</button>
          <button onclick="window.TileTrailsAuth.changeName()">✏️ CHANGE PLAYER NAME</button>
          <button class="danger" onclick="window.TileTrailsAuth.logout()">🚪 LOG OUT</button>
        </div>`;
    } else {
      renderAuthForm("login");
    }
    var modal = $("accountModal");
    if (modal) modal.classList.remove("hidden");
  }

  function closeAccount() {
    var modal = $("accountModal");
    if (modal) modal.classList.add("hidden");
  }

  async function ensureProfile(user) {
    var ref = db.collection("users").doc(user.uid);
    var snap = await ref.get();
    var d = snap.exists ? (snap.data() || {}) : {};
    var name = safeName(d.name || user.displayName || fallbackName(user)) || "Player";
    var pid = String(d.playerId || "").trim();
    if (!/^TT-[0-9]{8}$/.test(pid)) pid = makePlayerId();

    await ref.set({
      name: name,
      playerId: pid,
      email: user.email || "",
      updatedAt: firebase.firestore.FieldValue.serverTimestamp()
    }, { merge: true });

    profile = { name: name, playerId: pid };
    localStorage.setItem("tt_player_id", pid);
    localStorage.setItem("tt_player_name", name);
    setAccountView();
  }

  function gameState() {
    try {
      if (window.TileTrailsGame && typeof window.TileTrailsGame.getState === "function") return window.TileTrailsGame.getState();
    } catch (_) {}
    return {
      level: Number(localStorage.getItem("tt_level") || 1),
      maxUnlocked: Number(localStorage.getItem("tt_max_unlocked") || localStorage.getItem("tt_level") || 1),
      coins: Number(localStorage.getItem("tt_coins") || 0),
      streak: Number(localStorage.getItem("tt_streak") || 0),
      pu: JSON.parse(localStorage.getItem("tt_pu") || '{"undo":3,"shuffle":3,"hint":3}')
    };
  }

  function applyCloud(data) {
    if (!data) return;
    try {
      if (window.TileTrailsGame && typeof window.TileTrailsGame.applyCloudState === "function") {
        window.TileTrailsGame.applyCloudState(data);
        return;
      }
      ["level", "maxUnlocked", "coins", "streak"].forEach(function (k) {
        if (data[k] != null) localStorage.setItem("tt_" + (k === "maxUnlocked" ? "max_unlocked" : k), String(data[k]));
      });
      if (data.pu) localStorage.setItem("tt_pu", JSON.stringify(data.pu));
    } catch (e) { console.warn("Cloud state apply failed", e); }
  }

  async function saveCloud() {
    if (!ready || !currentUser) return;
    var data = gameState();
    data.maxUnlocked = Math.max(Number(data.maxUnlocked || 1), Number(data.level || 1));
    data.name = profile.name;
    data.playerId = profile.playerId;
    data.updatedAt = firebase.firestore.FieldValue.serverTimestamp();
    await db.collection("users").doc(currentUser.uid).set(data, { merge: true });
  }

  function queueSave() {
    clearTimeout(saveTimer);
    if (!currentUser) return;
    saveTimer = setTimeout(function () {
      saveCloud().catch(function (e) { console.warn("Cloud save failed", e); });
    }, 700);
  }

  async function loadCloud() {
    var snap = await db.collection("users").doc(currentUser.uid).get();
    if (!snap.exists) { await saveCloud(); return; }
    var d = snap.data() || {};
    var local = gameState();
    if (Number(local.maxUnlocked || 1) > Number(d.maxUnlocked || 1)) d.maxUnlocked = Number(local.maxUnlocked);
    applyCloud(d);
    if (d.name || d.playerId) {
      profile.name = safeName(d.name) || profile.name;
      profile.playerId = String(d.playerId || profile.playerId);
      setAccountView();
    }
  }

  async function login() {
    try {
      setStatus("Signing in…");
      await auth.signInWithEmailAndPassword($("authEmail").value.trim(), $("authPassword").value);
      closeAccount();
    } catch (e) { setStatus(firebaseError(e), true); }
  }

  function renderVerificationSent(email, sendError) {
    var body = $("accountBody");
    if (!body) return;
    body.innerHTML = `
      <div class="reward-icon">✉️</div>
      <h2>Verify your email</h2>
      <p>We sent a verification link to <b>${escapeHtml(email)}</b>.</p>
      <p style="font-size:14px">Open that email and tap the verification link. Login will continue to use <b>email + password only</b>.</p>
      ${sendError ? `<div class="auth-error">Account was created, but the verification email could not be sent automatically. Please try again from your Firebase email settings.</div>` : ""}
      <button class="modal-action" id="resendVerification">RESEND EMAIL</button>
      <button class="secondary-action full" id="verificationDone">DONE</button>`;
    $("resendVerification").onclick = async function () {
      try {
        if (!auth.currentUser) throw new Error("Please login first.");
        await auth.currentUser.sendEmailVerification();
        setStatus("Verification email sent again.");
      } catch (e) { setStatus(firebaseError(e), true); }
    };
    $("verificationDone").onclick = function () { closeAccount(); };
  }

  async function signup() {
    try {
      var name = safeName($("signupName").value);
      var email = $("authEmail").value.trim();
      var pass = $("authPassword").value;
      if (!name) throw new Error("Player name is required.");
      if (pass.length < 6) throw new Error("Password must be at least 6 characters.");
      setStatus("Creating account…");
      var cred = await auth.createUserWithEmailAndPassword(email, pass);
      await cred.user.updateProfile({ displayName: name });
      await ensureProfile(cred.user);
      await saveCloud();
      var sendError = false;
      try { await cred.user.sendEmailVerification(); } catch (mailError) {
        sendError = true;
        console.warn("Verification email failed:", mailError);
      }
      renderVerificationSent(email, sendError);
      alert("Account created! Your Player ID is " + profile.playerId + "\n\nVerification email sent. Please verify your email before sharing the account.");
    } catch (e) { setStatus(firebaseError(e), true); }
  }

  async function changeName() {
    if (!currentUser) return;
    var n = prompt("Enter your new Player Name:", profile.name || "Player");
    if (n === null) return;
    var name = safeName(n);
    if (!name) return alert("Name cannot be empty.");
    try {
      await currentUser.updateProfile({ displayName: name });
      await db.collection("users").doc(currentUser.uid).set({ name: name }, { merge: true });
      profile.name = name;
      setAccountView();
      openAccount();
      await saveCloud();
    } catch (e) { alert(firebaseError(e)); }
  }

  async function logout() {
    try {
      await saveCloud();
      await auth.signOut();
      closeAccount();
    } catch (e) { alert(firebaseError(e)); }
  }

  async function syncNow() {
    try {
      await saveCloud();
      alert("☁️ Game progress saved to cloud.");
    } catch (e) { alert(firebaseError(e)); }
  }

  function guestHint() {
    closeAccount();
    alert("Guest mode is active. Login or Sign Up anytime to sync progress across devices.");
  }

  function firebaseError(e) {
    var c = e && e.code || "";
    var map = {
      "auth/invalid-email": "Invalid email address.",
      "auth/user-not-found": "No account found with this email.",
      "auth/wrong-password": "Incorrect password.",
      "auth/invalid-credential": "Incorrect email or password.",
      "auth/email-already-in-use": "This email is already registered. Please login.",
      "auth/weak-password": "Password must be at least 6 characters.",
      "auth/network-request-failed": "Network error. Check your internet connection.",
      "auth/too-many-requests": "Too many attempts. Please try again later.",
      "auth/missing-email": "Please enter your email address.",
      "auth/api-key-not-valid": "Firebase API key is not valid for this Firebase project.",
      "auth/unauthorized-domain": "This website is not in Firebase Authorized Domains.",
      "permission-denied": "Firestore permission denied. Check your Firestore rules."
    };
    return map[c] || (e && e.message) || "Something went wrong. Please try again.";
  }

  async function boot() {
    try {
      if (!window.firebase) throw new Error("Firebase SDK is not loaded. Keep the Firebase CDN scripts in index.html.");
      var config = cfg();
      if (!config.apiKey || !config.projectId || !config.appId) throw new Error("Firebase configuration is incomplete.");

      if (!firebase.apps || !firebase.apps.length) firebase.initializeApp(config);
      auth = firebase.auth();
      db = firebase.firestore();
      ready = true;

      auth.onAuthStateChanged(async function (u) {
        currentUser = u || null;
        if (!currentUser) {
          profile = { name: "", playerId: "" };
          setAccountView();
          return;
        }
        try {
          await ensureProfile(currentUser);
          await loadCloud();
        } catch (e) {
          console.error("Firebase player/cloud load failed:", e);
          setStatus(firebaseError(e), true);
        }
      });
    } catch (e) {
      console.error("Firebase boot failed:", e);
      setAccountView();
    }
  }

  window.TileTrailsAuth = {
    render: renderAuthForm,
    openAccount: openAccount,
    logout: logout,
    changeName: changeName,
    syncNow: syncNow,
    queueSave: queueSave,
    saveNow: saveCloud,
    guestHint: guestHint,
    getUser: function () { return currentUser; },
    getPlayer: function () { return profile; },
    getPlayerId: function () { return profile.playerId; }
  };

  document.addEventListener("DOMContentLoaded", function () {
    var accountBtn = $("accountBtn");
    var closeBtn = $("closeAccountModal");
    if (accountBtn) accountBtn.onclick = openAccount;
    if (closeBtn) closeBtn.onclick = closeAccount;
    boot();
  });
})();
