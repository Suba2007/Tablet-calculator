# Tablet-calculator
<!doctype html>
<html lang="en">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width,initial-scale=1" />
  <title>Tablet Rate Calculator</title>
  <meta name="description" content="Simple Tablet Rate Calculator - add tablet name, price and quantity to get totals."/>
  <style>
    :root{
      --bg:#fbfdff; --card:#fff; --muted:#6b7280; --accent:#0f172a;
      font-family: Inter, system-ui, -apple-system, "Segoe UI", Roboto, "Helvetica Neue", Arial;
    }
    html,body{height:100%;margin:0;background:linear-gradient(180deg,#eef2ff 0%, #fbfdff 100%);color:var(--accent);}
    .wrap{max-width:900px;margin:22px auto;padding:18px;}
    header{display:flex;gap:12px;align-items:center;margin-bottom:12px}
    h1{font-size:20px;margin:0}
    p.lead{margin:0;color:var(--muted);font-size:13px}
    .card{background:var(--card);border-radius:12px;padding:14px;box-shadow:0 6px 18px rgba(2,6,23,0.06);margin-bottom:14px}
    form{display:grid;grid-template-columns:repeat(12,1fr);gap:8px;align-items:end}
    label{font-size:12px;color:var(--muted);display:block;margin-bottom:4px}
    input[type=text], input[type=number]{
      width:100%;padding:10px;border-radius:8px;border:1px solid #e6edf3;font-size:14px;
      box-sizing:border-box;
    }
    .col-4{grid-column:span 4}
    .col-3{grid-column:span 3}
    .col-12{grid-column:span 12}
    button{padding:10px 12px;border-radius:10px;border:0;background:#0b72ef;color:#fff;font-weight:600;cursor:pointer}
    button.secondary{background:#f1f5f9;color:var(--accent);border:1px solid #e6edf3}
    table{width:100%;border-collapse:collapse;margin-top:12px}
    th,td{padding:10px;text-align:left;border-bottom:1px solid #f1f5f6;font-size:14px}
    .muted{color:var(--muted)}
    .right{text-align:right}
    .totalbar{display:flex;justify-content:space-between;align-items:center;margin-top:10px;padding:12px;border-radius:10px;background:linear-gradient(90deg,#fff,#fbfbff);font-weight:700}
    .small{font-size:12px;color:var(--muted)}
    .actions button{margin-left:6px;padding:6px 8px;border-radius:8px;font-size:13px}
    @media (max-width:640px){
      form{grid-template-columns:repeat(6,1fr)}
      .col-4{grid-column:span 6}
      .col-3{grid-column:span 3}
    }
  </style>
</head>
<body>
  <div class="wrap">
    <header>
      <div>
        <h1>Tablet Rate Calculator</h1>
        <p class="lead">Add tablet name, price per tablet and quantity — total shows instantly.</p>
      </div>
    </header>

    <section class="card" aria-labelledby="add-title">
      <h2 id="add-title" style="font-size:15px;margin:0 0 8px 0">Add / Quick Add</h2>
      <form id="itemForm" onsubmit="return false" autocomplete="off">
        <div class="col-4">
          <label for="name">Tablet name</label>
          <input id="name" type="text" placeholder="e.g. Paracetamol" required />
        </div>

        <div class="col-3">
          <label for="price">Price per tablet (₹)</label>
          <input id="price" type="number" step="0.01" min="0" placeholder="e.g. 1.73" required />
        </div>

        <div class="col-3">
          <label for="qty">Quantity</label>
          <input id="qty" type="number" step="1" min="1" placeholder="e.g. 15" required />
        </div>

        <div class="col-2" style="display:flex;align-items:flex-end;gap:8px">
          <button id="addBtn" type="button">Add</button>
          <button id="clearBtn" class="secondary" type="button">Clear</button>
        </div>

        <div class="col-12 small" id="preview" style="display:none;margin-top:6px">
          Item total: <span id="itemTotalDisplay">₹0.00</span>
        </div>
      </form>
    </section>

    <section class="card" aria-labelledby="list-title">
      <h2 id="list-title" style="font-size:15px;margin:0 0 8px 0">Items</h2>
      <div id="emptyMsg" class="muted small">No items yet — add a tablet to see totals.</div>

      <div id="tableWrap" style="display:none">
        <table id="itemsTable" aria-live="polite">
          <thead>
            <tr>
              <th>Tablet</th>
              <th>Price / tablet</th>
              <th>Qty</th>
              <th class="right">Total</th>
              <th>Actions</th>
            </tr>
          </thead>
          <tbody></tbody>
        </table>

        <div class="totalbar" role="status" aria-live="polite">
          <div>
            <div class="small">Grand total</div>
            <div id="grandTotal" style="font-size:18px">₹0.00</div>
          </div>
          <div>
            <button id="exportBtn" class="secondary" title="Copy CSV">Copy CSV</button>
            <button id="resetBtn" style="margin-left:10px">Reset</button>
          </div>
        </div>
      </div>
    </section>

    <footer style="margin-top:8px;text-align:center" class="small muted">
      Saved locally on this browser (will stay on this device). Use "Add to Home screen" from your browser menu for app-like access.
    </footer>
  </div>

  <script>
    // Utilities
    const qs = s => document.querySelector(s);
    const formatRupee = v => "₹" + Number(v).toLocaleString('en-IN', {minimumFractionDigits:2, maximumFractionDigits:2});

    // Elements
    const nameEl = qs("#name"), priceEl = qs("#price"), qtyEl = qs("#qty");
    const addBtn = qs("#addBtn"), clearBtn = qs("#clearBtn"), preview = qs("#preview");
    const itemTotalDisplay = qs("#itemTotalDisplay");
    const tableWrap = qs("#tableWrap"), itemsTableBody = qs("#itemsTable tbody");
    const emptyMsg = qs("#emptyMsg"), grandTotalEl = qs("#grandTotal");
    const resetBtn = qs("#resetBtn"), exportBtn = qs("#exportBtn");

    // Storage key
    const STORAGE_KEY = "tablet_calc_items_v1";
    let items = [];

    // Load saved
    function load() {
      try {
        const raw = localStorage.getItem(STORAGE_KEY);
        if (raw) items = JSON.parse(raw);
      } catch(e) { items = []; }
      render();
    }

    function save() {
      localStorage.setItem(STORAGE_KEY, JSON.stringify(items));
    }

    function calcItemTotal(price, qty) {
      return Number(price) * Number(qty);
    }

    function render() {
      // Table
      itemsTableBody.innerHTML = "";
      if (items.length === 0) {
        tableWrap.style.display = "none";
        emptyMsg.style.display = "block";
      } else {
        tableWrap.style.display = "block";
        emptyMsg.style.display = "none";
        items.forEach((it, idx) => {
          const tr = document.createElement("tr");
          tr.innerHTML = `
            <td>${escapeHtml(it.name)}</td>
            <td>${formatRupee(it.price)}</td>
            <td>${it.qty}</td>
            <td class="right">${formatRupee(it.total)}</td>
            <td class="actions">
              <button data-idx="${idx}" class="editBtn">Edit</button>
              <button data-idx="${idx}" class="delBtn">Delete</button>
            </td>
          `;
          itemsTableBody.appendChild(tr);
        });
      }

      // Grand total
      const grand = items.reduce((s,i)=>s + Number(i.total), 0);
      grandTotalEl.textContent = formatRupee(grand);
      save();
      attachRowListeners();
    }

    function attachRowListeners() {
      qsAll(".delBtn").forEach(b => b.onclick = (e) => {
        const idx = Number(e.target.dataset.idx);
        items.splice(idx,1);
        render();
      });
      qsAll(".editBtn").forEach(b => b.onclick = (e) => {
        const idx = Number(e.target.dataset.idx);
        const it = items[idx];
        // populate form for edit
        nameEl.value = it.name;
        priceEl.value = it.price;
        qtyEl.value = it.qty;
        preview.style.display = "block";
        itemTotalDisplay.textContent = formatRupee(it.total);
        // remove original
        items.splice(idx,1);
        render();
        nameEl.focus();
      });
    }

    // helpers for querySelectorAll
    function qsAll(sel){ return Array.from(document.querySelectorAll(sel)); }
    function escapeHtml(s){
      return String(s).replace(/[&<>"]/g, c => ({'&':'&amp;','<':'&lt;','>':'&gt;','"':'&quot;'}[c]));
    }

    // Events
    function updatePreview(){
      const p = parseFloat(priceEl.value || 0);
      const q = parseInt(qtyEl.value || 0);
      if (p > 0 && q > 0) {
        preview.style.display = "block";
        itemTotalDisplay.textContent = formatRupee(calcItemTotal(p,q));
      } else {
        preview.style.display = "none";
      }
    }

    priceEl.addEventListener("input", updatePreview);
    qtyEl.addEventListener("input", updatePreview);

    addBtn.addEventListener("click", () => {
      const name = nameEl.value.trim();
      const price = parseFloat(priceEl.value);
      const qty = parseInt(qtyEl.value, 10);

      if (!name) { alert("Enter tablet name"); nameEl.focus(); return; }
      if (!price || price < 0) { alert("Enter valid price per tablet"); priceEl.focus(); return; }
      if (!qty || qty < 1) { alert("Enter valid quantity"); qtyEl.focus(); return; }

      const total = calcItemTotal(price, qty);
      items.push({ name, price: Number(price.toFixed(2)), qty, total: Number(total.toFixed(2)) });

      // clear form
      nameEl.value = ""; priceEl.value = ""; qtyEl.value = "";
      preview.style.display = "none";
      render();
      nameEl.focus();
    });

    clearBtn.addEventListener("click", () => {
      nameEl.value = ""; priceEl.value = ""; qtyEl.value = "";
      preview.style.display = "none";
      nameEl.focus();
    });

    resetBtn.addEventListener("click", () => {
      if (!confirm("Clear all items?")) return;
      items = []; save(); render();
    });

    exportBtn.addEventListener("click", () => {
      if (items.length === 0) { alert("No items to export"); return; }
      // CSV
      const rows = [["Tablet name","Price per tablet (₹)","Qty","Total (₹)"], ...items.map(i=>[i.name,i.price,i.qty,i.total])];
      const csv = rows.map(r=>r.map(c => `"${String(c).replace(/"/g,'""')}"`).join(",")).join("\n");
      // copy to clipboard
      navigator.clipboard.writeText(csv).then(()=>{
        alert("CSV copied to clipboard. You can paste in a notes or sheet app.");
      }, ()=>{ alert("Unable to copy. Try selecting and copying manually."); });
    });

    // init
    load();
  </script>
</body>
</html>
