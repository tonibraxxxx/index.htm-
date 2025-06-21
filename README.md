<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0"/>
  <title>Dept Tracker - Karanis Shop</title>
  <script src="https://cdnjs.cloudflare.com/ajax/libs/jspdf/2.5.1/jspdf.umd.min.js" defer></script>

  <script type="module">
    import { initializeApp } from "https://www.gstatic.com/firebasejs/10.12.2/firebase-app.js";
    import { getAuth, onAuthStateChanged, signInWithPopup, GoogleAuthProvider, signOut } from "https://www.gstatic.com/firebasejs/10.12.2/firebase-auth.js";
    import { getFirestore, collection, getDocs, addDoc, updateDoc, deleteDoc, doc, query, where } from "https://www.gstatic.com/firebasejs/10.12.2/firebase-firestore.js";
    import { getAnalytics } from "https://www.gstatic.com/firebasejs/10.12.2/firebase-analytics.js";

    const firebaseConfig = {
      apiKey: "AIzaSyDLxqvW8gHOV6DuaUZy_jvTWRfACoLccU4",
      authDomain: "shopping-mart-126e0.firebaseapp.com",
      projectId: "shopping-mart-126e0",
      storageBucket: "shopping-mart-126e0.firebasestorage.app",
      messagingSenderId: "1007757196380",
      appId: "1:1007757196380:web:4ff7d002521032ba167a99",
      measurementId: "G-6V079YZ13T"
    };

    const app = initializeApp(firebaseConfig);
    const analytics = getAnalytics(app);
    const auth = getAuth(app);
    const provider = new GoogleAuthProvider();
    const db = getFirestore(app);

    const loginBtn = document.createElement('button');
    loginBtn.textContent = "Login with Google";
    loginBtn.className = "btn-purple text-white px-4 py-2 my-4";

    const logoutBtn = document.createElement('button');
    logoutBtn.textContent = "Logout";
    logoutBtn.className = "btn-red text-white px-4 py-2 my-4";

    const downloadBtn = document.createElement('button');
    downloadBtn.textContent = "Download PDF";
    downloadBtn.className = "btn-pink text-white px-4 py-2 my-4";
    downloadBtn.addEventListener('click', downloadPDF);

    const main = document.querySelector('main');
    main.insertBefore(loginBtn, main.firstChild);

    let user = null;

    loginBtn.addEventListener('click', async () => {
      try {
        const result = await signInWithPopup(auth, provider);
        user = result.user;
        renderAuthUI();
        renderForm();
        renderList();
      } catch (error) {
        alert("Login failed: " + error.message);
      }
    });

    logoutBtn.addEventListener('click', async () => {
      await signOut(auth);
      location.reload();
    });

    onAuthStateChanged(auth, (currentUser) => {
      if (currentUser) {
        user = currentUser;
        renderAuthUI();
        renderForm();
        renderList();
      }
    });

    function renderAuthUI() {
      loginBtn.remove();
      main.insertBefore(logoutBtn, main.firstChild);
      main.insertBefore(downloadBtn, logoutBtn);
      const welcome = document.createElement('p');
      welcome.textContent = `Welcome, ${user.displayName}`;
      welcome.className = "font-bold mb-2";
      main.insertBefore(welcome, logoutBtn);
    }

    function renderForm() {
      const form = document.createElement('form');
      form.id = 'debtorForm';
      form.className = 'space-y-3 my-6';
      form.innerHTML = `
        <input id="name" placeholder="Name" required class="block w-full p-2 rounded" />
        <input id="mpesa" placeholder="MPesa Number" required class="block w-full p-2 rounded" />
        <input id="item" placeholder="Item" required class="block w-full p-2 rounded" />
        <input id="price" placeholder="Price" type="number" required class="block w-full p-2 rounded" />
        <input id="qty" placeholder="Quantity" type="number" required class="block w-full p-2 rounded" />
        <button type="submit" class="btn-purple text-white px-4 py-2">Add Debtor</button>
      `;

      main.insertBefore(form, document.getElementById('totalOwed'));

      form.addEventListener('submit', async (event) => {
        event.preventDefault();
        const name = document.getElementById('name').value.trim();
        const mpesa = document.getElementById('mpesa').value.trim();
        const item = document.getElementById('item').value.trim();
        const price = parseFloat(document.getElementById('price').value);
        const qty = parseInt(document.getElementById('qty').value);
        const total = price * qty;

        if (!name || !mpesa || !item || !price || !qty) return alert("Fill in all fields");

        await addDoc(collection(db, "debtors"), {
          uid: user.uid,
          name,
          mpesa,
          item,
          price,
          qty,
          total,
          paid: false
        });

        form.reset();
        renderList();
      });
    }

    async function renderList() {
      if (!user) return;
      const container = document.getElementById('debtorsList');
      container.innerHTML = '';

      const q = query(collection(db, "debtors"), where("uid", "==", user.uid));
      const snapshot = await getDocs(q);
      let totalOwed = 0;

      snapshot.forEach(docSnap => {
        const debtor = docSnap.data();
        const id = docSnap.id;
        if (!debtor.paid) totalOwed += debtor.total;

        const el = document.createElement('article');
        el.className = "border p-4 rounded-xl bg-white text-black shadow-md space-y-2";
        el.innerHTML = `
          <p><strong>Name:</strong> ${debtor.name}</p>
          <p><strong>MPesa:</strong> ${debtor.mpesa}</p>
          <p><strong>Items:</strong> ${debtor.item}</p>
          <p><strong>Quantity:</strong> ${debtor.qty}</p>
          <p><strong>Total:</strong> ${debtor.total} KES</p>
          <p><strong>Status:</strong> ${debtor.paid ? '✅ Paid' : '❌ Unpaid'}</p>
          <div class="flex flex-wrap gap-2">
            <button class="btn-pink px-3 py-1 text-white" onclick="togglePaid('${id}')">Mark as ${debtor.paid ? 'Unpaid' : 'Paid'}</button>
            <button class="btn-purple px-3 py-1 text-white" onclick="editDebtor('${id}', '${debtor.name}', '${debtor.mpesa}', '${debtor.item}', ${debtor.price}, ${debtor.qty})">Edit</button>
            <button class="btn-red px-3 py-1 text-white" onclick="deleteDebtor('${id}')">Delete</button>
          </div>
        `;
        container.appendChild(el);
      });

      const totalEl = document.getElementById('totalOwed');
      if (totalEl) totalEl.innerText = `Total Owed: ${totalOwed} KES`;
    }

    window.togglePaid = async function(id) {
      const ref = doc(db, "debtors", id);
      const snapshot = await getDocs(query(collection(db, "debtors"), where("uid", "==", user.uid)));
      snapshot.forEach(async snap => {
        if (snap.id === id) {
          await updateDoc(ref, { paid: !snap.data().paid });
          renderList();
        }
      });
    }

    window.deleteDebtor = async function(id) {
      await deleteDoc(doc(db, "debtors", id));
      renderList();
    }

    window.editDebtor = function(id, name, mpesa, item, price, qty) {
      document.getElementById('name').value = name;
      document.getElementById('mpesa').value = mpesa;
      document.getElementById('item').value = item;
      document.getElementById('price').value = price;
      document.getElementById('qty').value = qty;
      deleteDebtor(id);
    }

    async function downloadPDF() {
      const { jsPDF } = window.jspdf;
      const doc = new jsPDF();
      let y = 10;
      doc.text("Karanis Shop - Debtors List", 10, y);
      y += 10;
      const q = query(collection(db, "debtors"), where("uid", "==", user.uid));
      const snapshot = await getDocs(q);

      snapshot.forEach(debtorDoc => {
        const debtor = debtorDoc.data();
        doc.text(`Name: ${debtor.name}`, 10, y);
        doc.text(`MPesa: ${debtor.mpesa}`, 10, y + 10);
        doc.text(`Items: ${debtor.item}`, 10, y + 20);
        doc.text(`Total: ${debtor.total} KES`, 10, y + 30);
        doc.text(`Status: ${debtor.paid ? 'Paid' : 'Unpaid'}`, 10, y + 40);
        y += 50;
      });
      doc.save("debtors_list.pdf");
    }
  </script>

  <style>
    body {
      font-family: Arial, sans-serif;
      background-color: #eae6f7;
      color: #0f111d;
      padding: 20px;
    }
    .container {
      max-width: 600px;
      margin: 0 auto;
    }
    .btn-purple, .btn-red, .btn-pink {
      border: none;
      cursor: pointer;
      border-radius: 8px;
      box-shadow: 0 4px 6px rgba(0, 0, 0, 0.2);
    }
    .btn-purple { background-color: purple; }
    .btn-red { background-color: red; }
    .btn-pink { background-color: deeppink; }
  </style>
</head>
<body>
  <main class="container">
    <h1>Karanis Shop - Debtor Tracker</h1>
    <div id="totalOwed" class="mt-4 font-bold text-lg"></div>
    <section id="debtorsList" class="mt-4 space-y-4"></section>
  </main>
</body>
</html>
