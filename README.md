# convenienciaf
import os, sqlite3, json, base64, mimetypes, hashlib, secrets, socket, html
from http.server import ThreadingHTTPServer, BaseHTTPRequestHandler
from urllib.parse import urlparse, parse_qs
from datetime import datetime, date, timedelta

BASE = os.path.dirname(os.path.abspath(__file__))
DB = os.path.join(BASE, 'nosso_sonho.db')
UPLOADS = os.path.join(BASE, 'static', 'uploads')
os.makedirs(UPLOADS, exist_ok=True)
SESSIONS = {}

APP_NAME = 'CONVENIÊNCIA IRMÃO METRARIA'
APP_SUB = 'Nosso Sonho'

DEFAULT_USERS = [
    ('eryck','Eryck','0504','admin'),
    ('antonio','Antonio','123','operador'),
    ('pedro','Pedro','123','operador'),
]

def now(): return datetime.now().isoformat(timespec='seconds')
def today(): return date.today().isoformat()
def money(v): return round(float(v or 0),2)
def hash_pw(p): return hashlib.sha256(p.encode('utf-8')).hexdigest()

def db():
    c=sqlite3.connect(DB)
    c.row_factory=sqlite3.Row
    c.execute('PRAGMA foreign_keys=ON')
    return c

def init_db():
    c=db()
    c.executescript('''
    CREATE TABLE IF NOT EXISTS users(
      id INTEGER PRIMARY KEY AUTOINCREMENT, username TEXT UNIQUE NOT NULL, display_name TEXT NOT NULL,
      password_hash TEXT NOT NULL, role TEXT NOT NULL DEFAULT 'operador', active INTEGER DEFAULT 1, created_at TEXT NOT NULL
    );
    CREATE TABLE IF NOT EXISTS settings(k TEXT PRIMARY KEY, v TEXT);
    CREATE TABLE IF NOT EXISTS products(
      id INTEGER PRIMARY KEY AUTOINCREMENT, name TEXT NOT NULL, barcode TEXT UNIQUE, category TEXT,
      cost REAL DEFAULT 0, price REAL NOT NULL DEFAULT 0, stock REAL DEFAULT 0, min_stock REAL DEFAULT 0,
      image TEXT, active INTEGER DEFAULT 1, created_at TEXT NOT NULL
    );
    CREATE TABLE IF NOT EXISTS customers(
      id INTEGER PRIMARY KEY AUTOINCREMENT, name TEXT NOT NULL, phone TEXT, cpf_cnpj TEXT, address TEXT,
      created_at TEXT NOT NULL
    );
    CREATE TABLE IF NOT EXISTS suppliers(
      id INTEGER PRIMARY KEY AUTOINCREMENT, name TEXT NOT NULL, phone TEXT, cpf_cnpj TEXT, notes TEXT,
      created_at TEXT NOT NULL
    );
    CREATE TABLE IF NOT EXISTS cash_sessions(
      id INTEGER PRIMARY KEY AUTOINCREMENT, opened_by INTEGER, opened_at TEXT NOT NULL, opening_amount REAL DEFAULT 0,
      closed_at TEXT, closing_amount REAL, expected_amount REAL, difference REAL,
      FOREIGN KEY(opened_by) REFERENCES users(id)
    );
    CREATE TABLE IF NOT EXISTS sales(
      id INTEGER PRIMARY KEY AUTOINCREMENT, total REAL NOT NULL, payment TEXT NOT NULL, received REAL DEFAULT 0,
      change_amount REAL DEFAULT 0, user_id INTEGER, customer_id INTEGER, created_at TEXT NOT NULL,
      FOREIGN KEY(user_id) REFERENCES users(id), FOREIGN KEY(customer_id) REFERENCES customers(id)
    );
    CREATE TABLE IF NOT EXISTS sale_items(
      id INTEGER PRIMARY KEY AUTOINCREMENT, sale_id INTEGER NOT NULL, product_id INTEGER, name TEXT NOT NULL,
      qty REAL NOT NULL, unit_price REAL NOT NULL, subtotal REAL NOT NULL,
      FOREIGN KEY(sale_id) REFERENCES sales(id) ON DELETE CASCADE,
      FOREIGN KEY(product_id) REFERENCES products(id)
    );
    CREATE TABLE IF NOT EXISTS cash_movements(
      id INTEGER PRIMARY KEY AUTOINCREMENT, kind TEXT NOT NULL, amount REAL NOT NULL, description TEXT,
      user_id INTEGER, created_at TEXT NOT NULL,
      FOREIGN KEY(user_id) REFERENCES users(id)
    );
    CREATE TABLE IF NOT EXISTS stock_movements(
      id INTEGER PRIMARY KEY AUTOINCREMENT, product_id INTEGER, kind TEXT NOT NULL, qty REAL NOT NULL,
      reason TEXT, user_id INTEGER, created_at TEXT NOT NULL,
      FOREIGN KEY(product_id) REFERENCES products(id), FOREIGN KEY(user_id) REFERENCES users(id)
    );
    CREATE TABLE IF NOT EXISTS expenses(
      id INTEGER PRIMARY KEY AUTOINCREMENT, description TEXT NOT NULL, category TEXT, amount REAL NOT NULL,
      due_date TEXT, paid INTEGER DEFAULT 1, created_at TEXT NOT NULL, user_id INTEGER,
      FOREIGN KEY(user_id) REFERENCES users(id)
    );
    CREATE TABLE IF NOT EXISTS purchases(
      id INTEGER PRIMARY KEY AUTOINCREMENT, supplier_id INTEGER, total REAL DEFAULT 0, notes TEXT,
      created_at TEXT NOT NULL, user_id INTEGER,
      FOREIGN KEY(supplier_id) REFERENCES suppliers(id), FOREIGN KEY(user_id) REFERENCES users(id)
    );
    CREATE TABLE IF NOT EXISTS purchase_items(
      id INTEGER PRIMARY KEY AUTOINCREMENT, purchase_id INTEGER NOT NULL, product_id INTEGER,
      qty REAL NOT NULL, unit_cost REAL NOT NULL, subtotal REAL NOT NULL,
      FOREIGN KEY(purchase_id) REFERENCES purchases(id) ON DELETE CASCADE,
      FOREIGN KEY(product_id) REFERENCES products(id)
    );
    ''')
    for u,d,p,r in DEFAULT_USERS:
        if not c.execute('SELECT 1 FROM users WHERE username=?',(u,)).fetchone():
            c.execute('INSERT INTO users(username,display_name,password_hash,role,created_at) VALUES(?,?,?,?,?)',(u,d,hash_pw(p),r,now()))
    defaults={'company_name':APP_NAME,'system_name':APP_SUB,'cnpj':'','ie':'','address':'','phone':'','nfc_series':'1','nfc_csc':'','fiscal_note':'NFC-e/NF-e: configurar integração SEFAZ/provedor'}
    for k,v in defaults.items(): c.execute('INSERT OR IGNORE INTO settings(k,v) VALUES(?,?)',(k,v))
    c.commit(); c.close()

def settings(c): return {r['k']:r['v'] for r in c.execute('SELECT k,v FROM settings')}

def get_user(handler):
    token=handler.headers.get('Cookie','')
    token=next((x.split('=',1)[1] for x in token.split(';') if x.strip().startswith('session=')),None)
    return SESSIONS.get(token)

def require_user(handler):
    u=get_user(handler)
    if not u: handler.send_json({'error':'Não autenticado.'},401); return None
    return u

HTML = r'''<!doctype html><html lang="pt-BR"><head><meta charset="utf-8"><meta name="viewport" content="width=device-width,initial-scale=1"><meta name="theme-color" content="#111827"><title>CONVENIÊNCIA IRMÃO METRARIA</title>
<style>
*{box-sizing:border-box}body{margin:0;font-family:Inter,Arial,sans-serif;background:#f4f5f7;color:#17202a}button,input,select,textarea{font:inherit}button{cursor:pointer}header{height:62px;background:#111827;color:#fff;display:flex;align-items:center;justify-content:space-between;padding:0 18px;position:sticky;top:0;z-index:20}.brand{font-weight:800;letter-spacing:.3px}.brand small{display:block;font-weight:500;color:#cbd5e1;font-size:11px}.userbox{display:flex;gap:10px;align-items:center;font-size:13px}.wrap{display:flex;min-height:calc(100vh - 62px)}aside{width:235px;background:#1f2937;color:#fff;padding:12px;position:sticky;top:62px;height:calc(100vh - 62px);overflow:auto}aside button{width:100%;background:transparent;border:0;color:#e5e7eb;text-align:left;padding:11px;border-radius:9px;margin-bottom:3px}aside button:hover,aside button.active{background:#374151}.main{flex:1;padding:18px;max-width:1600px;width:100%;margin:auto}.page{display:none}.page.active{display:block}.cards{display:grid;grid-template-columns:repeat(5,1fr);gap:12px}.card,.panel{background:#fff;border-radius:13px;box-shadow:0 2px 9px #0000000d;padding:15px}.card .num{font-size:25px;font-weight:800;margin-top:5px}.grid2{display:grid;grid-template-columns:1fr 1fr;gap:14px}.grid3{display:grid;grid-template-columns:repeat(3,1fr);gap:12px}.row{display:flex;gap:8px;align-items:end}.row>*{flex:1}.formgrid{display:grid;grid-template-columns:repeat(2,1fr);gap:8px}.formgrid .full{grid-column:1/-1}label{font-size:12px;color:#4b5563;display:block}input,select,textarea{width:100%;padding:9px 10px;border:1px solid #d1d5db;border-radius:9px;background:#fff;margin:4px 0 9px}textarea{min-height:75px;resize:vertical}.btn{border:0;border-radius:9px;padding:9px 12px;background:#111827;color:#fff}.btn.green{background:#15803d}.btn.red{background:#b91c1c}.btn.blue{background:#2563eb}.btn.gray{background:#6b7280}.btn.orange{background:#c2410c}.btn.small{padding:6px 8px;font-size:12px}.muted{color:#6b7280}.danger{color:#b91c1c}.ok{color:#15803d}.tablewrap{overflow:auto}table{width:100%;border-collapse:collapse}th,td{padding:8px;border-bottom:1px solid #edf0f2;text-align:left;font-size:13px;vertical-align:middle}.thumb{width:48px;height:48px;object-fit:cover;border-radius:8px;background:#eee}.product-list{max-height:510px;overflow:auto}.prod{display:flex;gap:9px;align-items:center;padding:8px;border-bottom:1px solid #eee;cursor:pointer}.prod:hover{background:#f8fafc}.prod .grow{flex:1}.badge{padding:3px 7px;border-radius:999px;background:#fee2e2;color:#991b1b;font-size:11px}.badge.ok{background:#dcfce7;color:#166534}.cartrow{display:grid;grid-template-columns:1fr 65px 90px 28px;gap:6px;align-items:center;margin:6px 0}.cartrow input{margin:0}.total{font-size:27px;font-weight:800;text-align:right;margin:13px 0}.kpi{display:flex;justify-content:space-between;padding:8px 0;border-bottom:1px solid #eee}.reportpre{background:#111827;color:#f9fafb;padding:12px;border-radius:9px;white-space:pre-wrap}.login{min-height:100vh;display:grid;place-items:center;background:linear-gradient(135deg,#111827,#374151)}.loginbox{width:min(410px,92vw);background:#fff;padding:25px;border-radius:18px;box-shadow:0 15px 50px #0005}.loginbox h1{margin:0 0 3px}.imgpreview{max-height:120px;max-width:120px;border-radius:9px;margin:5px 0}.searchbar{display:flex;gap:7px}.searchbar input{margin:0}.sectiontitle{display:flex;justify-content:space-between;align-items:center;gap:10px}.hidden{display:none!important}.notice{padding:10px;border-radius:9px;background:#eff6ff;color:#1e40af;margin-bottom:10px}.modal{position:fixed;inset:0;background:#0008;display:grid;place-items:center;z-index:100}.modalbox{background:white;width:min(650px,94vw);max-height:90vh;overflow:auto;border-radius:14px;padding:18px}.modalhead{display:flex;justify-content:space-between}.pill{display:inline-block;padding:3px 7px;background:#eef2ff;color:#3730a3;border-radius:99px;font-size:11px}
@media(max-width:1000px){aside{width:190px}.cards{grid-template-columns:repeat(3,1fr)}}@media(max-width:760px){.wrap{display:block}aside{position:sticky;top:62px;width:100%;height:auto;display:flex;overflow:auto;gap:4px;padding:7px;z-index:15}aside button{min-width:max-content;margin:0}.main{padding:10px}.cards{grid-template-columns:1fr 1fr}.grid2,.grid3{grid-template-columns:1fr}.formgrid{grid-template-columns:1fr}.row{display:grid;grid-template-columns:1fr 1fr}.userbox span{display:none}.cartrow{grid-template-columns:1fr 58px 78px 26px}}
</style></head><body><div id="app"></div>
<script>
const $=id=>document.getElementById(id), money=v=>new Intl.NumberFormat('pt-BR',{style:'currency',currency:'BRL'}).format(Number(v||0));
let products=[],cart=[],me=null,settings={};
async function api(path,opt={}){let r=await fetch(path,{headers:{'Content-Type':'application/json',...(opt.headers||{})},...opt});let d=await r.json().catch(()=>({error:'Resposta inválida'}));if(r.status===401){showLogin();throw new Error('auth')}return d}
function esc(s){return String(s??'').replace(/[&<>"']/g,m=>({'&':'&amp;','<':'&lt;','>':'&gt;','"':'&quot;',"'":'&#39;'}[m]))}
function showLogin(){document.body.innerHTML=`<div class="login"><div class="loginbox"><h1>🏪 CONVENIÊNCIA IRMÃO METRARIA</h1><p class="muted">${esc('Nosso Sonho')} · acesso ao sistema</p><form onsubmit="login(event)"><label>Login</label><input id="loginUser" autocomplete="username" required><label>Senha</label><input id="loginPass" type="password" autocomplete="current-password" required><button class="btn" style="width:100%;margin-top:6px">Entrar</button><p id="loginErr" class="danger"></p></form></div></div>`}
async function login(e){e.preventDefault();let d=await api('/api/login',{method:'POST',body:JSON.stringify({username:$('loginUser').value,password:$('loginPass').value})});if(d.error){$('loginErr').textContent=d.error;return}me=d.user;renderApp();}
function renderApp(){document.body.innerHTML=`<header><div class="brand">🏪 CONVENIÊNCIA IRMÃO METRARIA<small>Nosso Sonho · sistema de gestão</small></div><div class="userbox"><span id="clock"></span><span>👤 ${esc(me.display_name)}</span><button class="btn gray small" onclick="logout()">Sair</button></div></header><div class="wrap"><aside><button data-page="home">🏠 Início</button><button data-page="pdv">🛒 Caixa / PDV</button><button data-page="products">📦 Produtos</button><button data-page="stock">📥 Estoque / Compras</button><button data-page="customers">👥 Clientes</button><button data-page="suppliers">🚚 Fornecedores</button><button data-page="expenses">💸 Despesas</button><button data-page="reports">📊 Relatórios</button><button data-page="cash">💰 Caixa</button><button data-page="fiscal">🧾 Fiscal / Impressão</button><button data-page="settings">⚙️ Configurações</button></aside><main class="main">
<section id="home" class="page"><div class="sectiontitle"><h2>Dashboard</h2><button class="btn blue" onclick="loadDash()">Atualizar</button></div><div class="cards"><div class="card">Vendas hoje<div class="num" id="dSales">R$ 0,00</div></div><div class="card">Pedidos<div class="num" id="dCount">0</div></div><div class="card">Itens vendidos<div class="num" id="dItems">0</div></div><div class="card">Estoque baixo<div class="num" id="dLow">0</div></div><div class="card">Lucro estimado<div class="num" id="dProfit">R$ 0,00</div></div></div><div class="grid2" style="margin-top:14px"><div class="panel"><h3>Resumo do dia</h3><div id="dSummary"></div></div><div class="panel"><h3>Formas de pagamento</h3><div id="dPayments"></div></div></div></section>
<section id="pdv" class="page"><div class="sectiontitle"><h2>Caixa / PDV</h2><div><button class="btn orange" onclick="openCash()">Abrir caixa</button> <button class="btn gray" onclick="closeCash()">Fechar caixa</button></div></div><div id="cashStatus" class="notice">Carregando caixa...</div><div class="grid2"><div class="panel"><h3>Produtos</h3><div class="searchbar"><input id="pdvSearch" placeholder="Nome ou código de barras" autofocus><button class="btn blue" onclick="scanLookup()">Buscar</button></div><div id="pdvList" class="product-list"></div></div><div class="panel"><h3>Venda atual</h3><div id="cart"></div><div class="total">Total: <span id="cartTotal">R$ 0,00</span></div><label>Cliente (opcional)</label><select id="saleCustomer"><option value="">Consumidor</option></select><label>Forma de pagamento</label><select id="payment"><option value="dinheiro">Dinheiro</option><option value="pix">Pix</option><option value="debito">Cartão de débito</option><option value="credito">Cartão de crédito</option></select><div id="cashBox"><label>Valor recebido</label><input id="received" type="number" step="0.01" min="0" placeholder="0,00"><div class="kpi"><b>Troco</b><b id="change">R$ 0,00</b></div></div><button class="btn green" style="width:100%;margin-top:10px" onclick="finishSale(true)">Finalizar e imprimir</button><button class="btn blue" style="width:100%;margin-top:7px" onclick="finishSale(false)">Finalizar sem imprimir</button><button class="btn gray" style="width:100%;margin-top:7px" onclick="clearCart()">Cancelar venda</button></div></div></section>
<section id="products" class="page"><div class="sectiontitle"><h2>Produtos</h2><button class="btn green" onclick="newProduct()">+ Novo produto</button></div><div class="grid2"><div class="panel"><h3 id="prodFormTitle">Cadastrar produto</h3><input type="hidden" id="pid"><div class="formgrid"><div><label>Nome</label><input id="pname"></div><div><label>Código de barras</label><input id="pbarcode"></div><div><label>Categoria</label><input id="pcat"></div><div><label>Custo</label><input id="pcost" type="number" step="0.01"></div><div><label>Preço de venda</label><input id="pprice" type="number" step="0.01"></div><div><label>Estoque</label><input id="pstock" type="number" step="0.01"></div><div><label>Estoque mínimo</label><input id="pmin" type="number" step="0.01"></div><div><label>Foto</label><input id="pimage" type="file" accept="image/*"></div></div><div id="pPreview"></div><button class="btn green" onclick="saveProduct()">Salvar produto</button> <button class="btn gray" onclick="newProduct()">Limpar</button></div><div class="panel"><div class="searchbar"><input id="prodFilter" placeholder="Pesquisar produto"><button class="btn blue" onclick="loadProducts()">Atualizar</button></div><div id="prodTable" class="tablewrap"></div></div></div></section>
<section id="stock" class="page"><div class="sectiontitle"><h2>Estoque / Compras</h2><button class="btn blue" onclick="loadStock()">Atualizar</button></div><div class="grid2"><div class="panel"><h3>Entrada / ajuste de estoque</h3><label>Produto</label><select id="stockProduct"></select><div class="row"><div><label>Quantidade</label><input id="stockQty" type="number" step="0.01"></div><div><label>Tipo</label><select id="stockKind"><option value="entrada">Entrada</option><option value="ajuste">Ajuste</option><option value="perda">Perda</option></select></div></div><label>Motivo</label><input id="stockReason" placeholder="Compra, quebra, perda..."><button class="btn green" onclick="stockMove()">Registrar movimentação</button></div><div class="panel"><h3>Estoque atual</h3><div id="stockTable" class="tablewrap"></div></div></div></section>
<section id="customers" class="page"><div class="sectiontitle"><h2>Clientes</h2><button class="btn green" onclick="newCustomer()">+ Cliente</button></div><div class="grid2"><div class="panel"><input type="hidden" id="cid"><input id="cname" placeholder="Nome"><input id="cphone" placeholder="Telefone"><input id="ccpf" placeholder="CPF/CNPJ"><input id="caddr" placeholder="Endereço"><button class="btn green" onclick="saveCustomer()">Salvar</button></div><div class="panel"><div id="customerTable" class="tablewrap"></div></div></div></section>
<section id="suppliers" class="page"><div class="sectiontitle"><h2>Fornecedores</h2><button class="btn green" onclick="newSupplier()">+ Fornecedor</button></div><div class="grid2"><div class="panel"><input type="hidden" id="sid"><input id="sname" placeholder="Nome"><input id="sphone" placeholder="Telefone"><input id="scpf" placeholder="CNPJ/CPF"><textarea id="snotes" placeholder="Observações"></textarea><button class="btn green" onclick="saveSupplier()">Salvar</button></div><div class="panel"><div id="supplierTable" class="tablewrap"></div></div></div></section>
<section id="expenses" class="page"><h2>Despesas</h2><div class="grid2"><div class="panel"><h3>Lançar despesa</h3><input id="edesc" placeholder="Descrição"><input id="ecat" placeholder="Categoria"><input id="eamount" type="number" step="0.01" placeholder="Valor"><input id="edue" type="date"><label><input id="epaid" type="checkbox" checked style="width:auto"> Paga</label><button class="btn green" onclick="saveExpense()">Lançar</button></div><div class="panel"><h3>Despesas recentes</h3><div id="expenseTable" class="tablewrap"></div></div></div></section>
<section id="reports" class="page"><div class="sectiontitle"><h2>Relatórios</h2><div><input id="reportDate" type="date" style="width:auto"><button class="btn blue" onclick="loadReport()">Consultar</button> <button class="btn gray" onclick="printArea('report')">Imprimir</button></div></div><div id="report"></div></section>
<section id="cash" class="page"><div class="sectiontitle"><h2>Controle de Caixa</h2><div><button class="btn orange" onclick="openCash()">Abrir</button> <button class="btn gray" onclick="closeCash()">Fechar</button></div></div><div class="grid2"><div class="panel"><h3>Movimentação manual</h3><div class="row"><select id="cashKind"><option value="entrada">Entrada</option><option value="saida">Saída</option></select><input id="cashAmount" type="number" step="0.01" placeholder="Valor"><input id="cashDesc" placeholder="Descrição"><button class="btn green" onclick="addCash()">Lançar</button></div></div><div class="panel"><h3>Caixa atual</h3><div id="cashSummary"></div></div></div><div class="panel"><h3>Movimentações</h3><div id="cashList" class="tablewrap"></div></div></section>
<section id="fiscal" class="page"><h2>Fiscal / Impressão</h2><div class="notice">O sistema imprime recibos/cupom não fiscal e relatórios. Para <b>NFC-e/NF-e com validade fiscal</b>, é necessário configurar CNPJ, IE, certificado digital e a integração com SEFAZ/provedor. Esta versão não fabrica documento fiscal.</div><div class="grid2"><div class="panel"><h3>Dados da empresa</h3><input id="fCompany" placeholder="Razão/nome"><input id="fCnpj" placeholder="CNPJ"><input id="fIe" placeholder="Inscrição Estadual"><input id="fPhone" placeholder="Telefone"><input id="fAddress" placeholder="Endereço"><input id="fSeries" placeholder="Série NFC-e"><input id="fCsc" placeholder="CSC/token (deixe vazio até configurar)"><button class="btn green" onclick="saveFiscal()">Salvar configurações</button></div><div class="panel"><h3>Impressão</h3><p>Compatível com impressão pelo navegador em impressoras térmicas 58/80 mm e impressora A4.</p><button class="btn blue" onclick="testPrint()">Imprimir teste</button><hr><p class="muted">Quando a integração fiscal for contratada/configurada, este módulo poderá enviar a NFC-e/NF-e para o provedor correspondente.</p></div></div></section>
<section id="settings" class="page"><div class="sectiontitle"><h2>Configurações</h2><span class="pill">${esc(me.role)}</span></div><div class="grid2"><div class="panel"><h3>Meu login</h3><label>Nome exibido</label><input id="myName"><label>Login</label><input id="myUser"><label>Senha atual</label><input id="oldPass" type="password"><label>Nova senha</label><input id="newPass" type="password"><button class="btn blue" onclick="changeMine()">Salvar meus dados</button></div><div class="panel"><h3>Usuários</h3><div id="usersTable"></div></div></div><div class="panel"><h3>Backup</h3><p>Faça backups frequentes do banco de dados para não perder vendas, estoque e cadastros.</p><button class="btn green" onclick="downloadBackup()">Baixar backup do sistema</button></div></section>
</main></div><div id="modal"></div>`;
document.querySelectorAll('aside button').forEach(b=>b.onclick=()=>page(b.dataset.page));page('home');setInterval(()=>{let c=document.getElementById('clock');if(c)c.textContent=new Date().toLocaleString('pt-BR')},1000);loadDash();loadProducts();loadCustomers();loadCashStatus();loadFiscal();loadUsers();}
function page(p){document.querySelectorAll('.page').forEach(x=>x.classList.remove('active'));$(p).classList.add('active');document.querySelectorAll('aside button').forEach(x=>x.classList.toggle('active',x.dataset.page===p));let fn={home:loadDash,pdv:loadPDV,products:loadProducts,stock:loadStock,customers:loadCustomers,suppliers:loadSuppliers,expenses:loadExpenses,reports:loadReport,cash:loadCash,fiscal:loadFiscal,settings:loadSettings}[p];if(fn)fn()}
async function logout(){await api('/api/logout',{method:'POST'});showLogin()}
async function loadProducts(){products=await api('/api/products');let q=(($('prodFilter')?.value)||'').toLowerCase();let a=products.filter(p=>(p.name+' '+(p.barcode||'')+' '+(p.category||'')).toLowerCase().includes(q));if($('prodTable'))$('prodTable').innerHTML='<table><tr><th></th><th>Produto</th><th>Código</th><th>Venda</th><th>Margem</th><th>Estoque</th><th></th></tr>'+a.map(p=>`<tr><td>${p.image?`<img class="thumb" src="${p.image}">`:'📦'}</td><td><b>${esc(p.name)}</b><br><span class="muted">${esc(p.category||'')}</span></td><td>${esc(p.barcode||'-')}</td><td>${money(p.price)}</td><td>${p.cost?(((p.price-p.cost)/p.price)*100).toFixed(1)+'%':'-'}</td><td>${p.stock<=p.min_stock?`<span class="badge">${p.stock}</span>`:p.stock}</td><td><button class="btn small blue" onclick="editProduct(${p.id})">Editar</button></td></tr>`).join('')+'</table>';if($('stockProduct'))$('stockProduct').innerHTML=products.map(p=>`<option value="${p.id}">${esc(p.name)} · estoque ${p.stock}</option>`).join('');renderPDVList()}
function renderPDVList(){if(!$('pdvList'))return;let q=(($('pdvSearch')?.value)||'').toLowerCase();let a=products.filter(p=>(p.name+' '+(p.barcode||'')).toLowerCase().includes(q));$('pdvList').innerHTML=a.slice(0,100).map(p=>`<div class="prod" onclick="addCart(${p.id})">${p.image?`<img class="thumb" src="${p.image}">`:'📦'}<div class="grow"><b>${esc(p.name)}</b><br><span class="muted">${esc(p.barcode||'')}</span></div><b>${money(p.price)}</b><span class="pill">${p.stock}</span></div>`).join('')||'<p class="muted">Nenhum produto.</p>'}
function scanLookup(){renderPDVList();let q=($('pdvSearch').value||'').trim();if(q){let p=products.find(x=>x.barcode===q);if(p){addCart(p.id);$('pdvSearch').value='';renderPDVList()}}}
$('pdvSearch')?.addEventListener('input',renderPDVList);$('pdvSearch')?.addEventListener('keydown',e=>{if(e.key==='Enter'){e.preventDefault();scanLookup()}})
function addCart(id){let p=products.find(x=>x.id===id);if(!p||p.stock<=0)return alert('Produto sem estoque.');let x=cart.find(i=>i.id===id);if(x){if(x.qty+1>p.stock)return alert('Estoque insuficiente.');x.qty++}else cart.push({id:p.id,name:p.name,price:p.price,qty:1});renderCart()}
function renderCart(){if(!$('cart'))return;$('cart').innerHTML=cart.length?cart.map((x,i)=>`<div class="cartrow"><span>${esc(x.name)}</span><input type="number" min="1" step="1" value="${x.qty}" onchange="setQty(${i},this.value)"><span>${money(x.price*x.qty)}</span><button onclick="cart.splice(${i},1);renderCart()">×</button></div>`).join(''):'<p class="muted">Carrinho vazio.</p>';let t=cart.reduce((s,x)=>s+x.price*x.qty,0);$('cartTotal').textContent=money(t);calcChange()}
function setQty(i,v){let p=products.find(x=>x.id===cart[i].id),q=Math.max(1,+v||1);if(q>p.stock)return alert('Estoque insuficiente.');cart[i].qty=q;renderCart()}
function clearCart(){cart=[];renderCart()}
function calcChange(){if(!$('change'))return;let t=cart.reduce((s,x)=>s+x.price*x.qty,0),r=+$('received').value||0;$('change').textContent=money(Math.max(0,r-t))}
function totalCart(){return money(cart.reduce((s,x)=>s+x.price*x.qty,0))}
async function loadPDV(){await loadProducts();await loadCustomers();loadCashStatus();renderCart();if($('payment'))$('payment').onchange=()=>{$('cashBox').style.display=$('payment').value==='dinheiro'?'block':'none'};if($('received'))$('received').oninput=calcChange}
async function finishSale(print){if(!cart.length)return alert('Carrinho vazio.');let total=cart.reduce((s,x)=>s+x.price*x.qty,0),payment=$('payment').value,received=payment==='dinheiro'?(+$('received').value||0):total;if(payment==='dinheiro'&&received<total)return alert('Valor recebido menor que o total.');let d=await api('/api/sales',{method:'POST',body:JSON.stringify({items:cart,payment,received,customer_id:$('saleCustomer').value||null})});if(d.error)return alert(d.error);if(print)printReceipt(d.sale);cart=[];renderCart();loadProducts();loadDash();}
function printReceipt(s){let w=window.open('','_blank','width=420,height=720');if(!w)return alert('Permita pop-ups para imprimir.');let company=(settings.company_name||APP_NAME);w.document.write(`<html><head><title>Cupom ${s.id}</title><style>body{font-family:monospace;width:300px;margin:0 auto;font-size:12px}h2{text-align:center;margin-bottom:3px}.line{border-top:1px dashed #000;margin:8px 0}p{margin:4px 0}.r{text-align:right}</style></head><body><h2>${esc(company)}</h2><div style="text-align:center">RECIBO / CUPOM NÃO FISCAL</div><div class=line></div>${s.items.map(i=>`<p>${i.qty}x ${esc(i.name)}<br><span class=r>${money(i.subtotal)}</span></p>`).join('')}<div class=line></div><p><b>TOTAL: ${money(s.total)}</b></p><p>PAGAMENTO: ${esc(s.payment)}</p>${s.payment==='dinheiro'?`<p>RECEBIDO: ${money(s.received)}<br>TROCO: ${money(s.change_amount)}</p>`:''}<p>Venda #${s.id}<br>${new Date(s.created_at).toLocaleString('pt-BR')}</p><div class=line></div><p style="text-align:center">Obrigado pela preferência!</p><script>window.print();setTimeout(()=>window.close(),500)<\/script></body></html>`);w.document.close()}
async function openCash(){let amount=prompt('Valor inicial do caixa (R$):','0');if(amount===null)return;let d=await api('/api/cash/open',{method:'POST',body:JSON.stringify({amount:+amount||0})});if(d.error)return alert(d.error);alert('Caixa aberto.');loadCashStatus()}
async function closeCash(){let amount=prompt('Digite o dinheiro contado no fechamento (R$):');if(amount===null)return;let d=await api('/api/cash/close',{method:'POST',body:JSON.stringify({amount:+amount||0})});if(d.error)return alert(d.error);alert(`Caixa fechado. Esperado: ${money(d.expected)} · Contado: ${money(d.counted)} · Diferença: ${money(d.difference)}`);loadCashStatus();loadCash()}
async function loadCashStatus(){if(!$('cashStatus'))return;let d=await api('/api/cash/status');$('cashStatus').innerHTML=d.open?`<b>Caixa aberto</b> desde ${new Date(d.opened_at).toLocaleString('pt-BR')} · abertura ${money(d.opening_amount)} · esperado ${money(d.expected)}`:'<b>Caixa fechado.</b> Abra o caixa antes de vender.'}
async function saveProduct(){let id=$('pid').value;let img='';let f=$('pimage').files[0];if(f){img=await fileData(f)}else if(id){img=$('pPreview').dataset.current||''}let body={name:$('pname').value,barcode:$('pbarcode').value,category:$('pcat').value,cost:+$('pcost').value||0,price:+$('pprice').value||0,stock:+$('pstock').value||0,min_stock:+$('pmin').value||0,image:img};let d=await api(id?'/api/products/'+id:'/api/products',{method:'POST',body:JSON.stringify(body)});if(d.error)return alert(d.error);newProduct();loadProducts();}
function fileData(f){return new Promise((res,rej)=>{let r=new FileReader();r.onload=()=>res(r.result);r.onerror=rej;r.readAsDataURL(f)})}
async function editProduct(id){let p=products.find(x=>x.id===id);if(!p)return;$('pid').value=p.id;$('pname').value=p.name;$('pbarcode').value=p.barcode||'';$('pcat').value=p.category||'';$('pcost').value=p.cost;$('pprice').value=p.price;$('pstock').value=p.stock;$('pmin').value=p.min_stock;$('pPreview').dataset.current=p.image||'';$('pPreview').innerHTML=p.image?`<img class="imgpreview" src="${p.image}">`:'';$('prodFormTitle').textContent='Editar produto';window.scrollTo({top:0,behavior:'smooth'})}
function newProduct(){['pid','pname','pbarcode','pcat','pcost','pprice','pstock','pmin'].forEach(id=>{if($(id))$(id).value=''});if($('pimage'))$('pimage').value='';if($('pPreview')){$('pPreview').innerHTML='';$('pPreview').dataset.current=''}if($('prodFormTitle'))$('prodFormTitle').textContent='Cadastrar produto'}
async function stockMove(){let d=await api('/api/stock',{method:'POST',body:JSON.stringify({product_id:+$('stockProduct').value,qty:+$('stockQty').value,kind:$('stockKind').value,reason:$('stockReason').value})});if(d.error)return alert(d.error);alert('Movimentação registrada.');$('stockQty').value='';$('stockReason').value='';loadProducts();loadStock()}
async function loadStock(){await loadProducts();if(!$('stockTable'))return;$('stockTable').innerHTML='<table><tr><th>Produto</th><th>Estoque</th><th>Mínimo</th><th>Status</th></tr>'+products.map(p=>`<tr><td>${esc(p.name)}</td><td>${p.stock}</td><td>${p.min_stock}</td><td>${p.stock<=p.min_stock?'<span class="badge">BAIXO</span>':'<span class="badge ok">OK</span>'}</td></tr>`).join('')+'</table>'}
async function loadCustomers(){let a=await api('/api/customers');if($('customerTable'))$('customerTable').innerHTML='<table><tr><th>Nome</th><th>Telefone</th><th>CPF/CNPJ</th></tr>'+a.map(x=>`<tr><td>${esc(x.name)}</td><td>${esc(x.phone||'')}</td><td>${esc(x.cpf_cnpj||'')}</td></tr>`).join('')+'</table>';if($('saleCustomer'))$('saleCustomer').innerHTML='<option value="">Consumidor</option>'+a.map(x=>`<option value="${x.id}">${esc(x.name)}</option>`).join('')}
function newCustomer(){['cid','cname','cphone','ccpf','caddr'].forEach(id=>$(id).value='')}
async function saveCustomer(){let d=await api('/api/customers',{method:'POST',body:JSON.stringify({name:$('cname').value,phone:$('cphone').value,cpf_cnpj:$('ccpf').value,address:$('caddr').value})});if(d.error)return alert(d.error);newCustomer();loadCustomers()}
async function loadSuppliers(){let a=await api('/api/suppliers');if($('supplierTable'))$('supplierTable').innerHTML='<table><tr><th>Nome</th><th>Telefone</th><th>CNPJ/CPF</th><th>Obs.</th></tr>'+a.map(x=>`<tr><td>${esc(x.name)}</td><td>${esc(x.phone||'')}</td><td>${esc(x.cpf_cnpj||'')}</td><td>${esc(x.notes||'')}</td></tr>`).join('')+'</table>'}
function newSupplier(){['sid','sname','sphone','scpf','snotes'].forEach(id=>$(id).value='')}
async function saveSupplier(){let d=await api('/api/suppliers',{method:'POST',body:JSON.stringify({name:$('sname').value,phone:$('sphone').value,cpf_cnpj:$('scpf').value,notes:$('snotes').value})});if(d.error)return alert(d.error);newSupplier();loadSuppliers()}
async function loadExpenses(){let a=await api('/api/expenses');if($('expenseTable'))$('expenseTable').innerHTML='<table><tr><th>Data</th><th>Descrição</th><th>Categoria</th><th>Valor</th><th>Status</th></tr>'+a.map(x=>`<tr><td>${new Date(x.created_at).toLocaleDateString('pt-BR')}</td><td>${esc(x.description)}</td><td>${esc(x.category||'')}</td><td>${money(x.amount)}</td><td>${x.paid?'Paga':'Pendente'}</td></tr>`).join('')+'</table>'}
async function saveExpense(){let d=await api('/api/expenses',{method:'POST',body:JSON.stringify({description:$('edesc').value,category:$('ecat').value,amount:+$('eamount').value,due_date:$('edue').value,paid:$('epaid').checked})});if(d.error)return alert(d.error);$('edesc').value=$('ecat').value=$('eamount').value=$('edue').value='';loadExpenses()}
async function loadDash(){let d=await api('/api/dashboard');if($('dSales')){$('dSales').textContent=money(d.sales);$('dCount').textContent=d.count;$('dItems').textContent=d.items;$('dLow').textContent=d.low;$('dProfit').textContent=money(d.profit);$('dSummary').innerHTML=`<div class="kpi"><span>Faturamento</span><b>${money(d.sales)}</b></div><div class="kpi"><span>Despesas pagas hoje</span><b>${money(d.expenses)}</b></div><div class="kpi"><span>Resultado estimado</span><b>${money(d.sales-d.expenses)}</b></div>`;$('dPayments').innerHTML=Object.entries(d.payments).map(([k,v])=>`<div class="kpi"><span>${esc(k)}</span><b>${money(v)}</b></div>`).join('')||'<span class="muted">Nenhuma venda hoje.</span>'}}
async function loadReport(){let d=await api('/api/report?date='+($('reportDate')?.value||today()));$('report').innerHTML=`<div class="grid3"><div class="card">Faturamento<div class="num">${money(d.total)}</div></div><div class="card">Vendas<div class="num">${d.count}</div></div><div class="card">Lucro bruto estimado<div class="num">${money(d.profit)}</div></div></div><div class="panel" style="margin-top:12px"><h3>Produtos vendidos em ${d.date.split('-').reverse().join('/')}</h3><div class="tablewrap"><table><tr><th>Produto</th><th>Quantidade</th><th>Vendas</th><th>Custo estimado</th><th>Lucro</th></tr>${d.products.map(x=>`<tr><td>${esc(x.name)}</td><td>${x.qty}</td><td>${money(x.total)}</td><td>${money(x.cost)}</td><td>${money(x.profit)}</td></tr>`).join('')}</table></div><hr><p><b>Formas de pagamento:</b> ${Object.entries(d.payments).map(([k,v])=>esc(k)+': '+money(v)).join(' · ')}</p></div>`}
function printArea(id){let content=$(id)?.innerHTML;if(!content)return;let w=window.open('','_blank');w.document.write('<html><head><title>Relatório</title><style>body{font-family:Arial;margin:20px}table{width:100%;border-collapse:collapse}td,th{padding:7px;border-bottom:1px solid #ddd}</style></head><body>'+content+'<script>window.print();setTimeout(()=>window.close(),500)<\/script></body></html>');w.document.close()}
async function loadCash(){let d=await api('/api/cash');if($('cashSummary'))$('cashSummary').innerHTML=`<div class="kpi"><span>Abertura</span><b>${money(d.opening)}</b></div><div class="kpi"><span>Vendas em dinheiro</span><b>${money(d.cashSales)}</b></div><div class="kpi"><span>Entradas</span><b>${money(d.entries)}</b></div><div class="kpi"><span>Saídas</span><b>${money(d.exits)}</b></div><div class="kpi"><span>Esperado</span><b>${money(d.expected)}</b></div>`;if($('cashList'))$('cashList').innerHTML='<table><tr><th>Hora</th><th>Tipo</th><th>Valor</th><th>Descrição</th></tr>'+d.movements.map(x=>`<tr><td>${new Date(x.created_at).toLocaleTimeString('pt-BR')}</td><td>${esc(x.kind)}</td><td>${money(x.amount)}</td><td>${esc(x.description||'Venda')}</td></tr>`).join('')+'</table>'}
async function addCash(){let d=await api('/api/cash',{method:'POST',body:JSON.stringify({kind:$('cashKind').value,amount:+$('cashAmount').value,description:$('cashDesc').value})});if(d.error)return alert(d.error);$('cashAmount').value='';$('cashDesc').value='';loadCash();loadCashStatus()}
async function loadFiscal(){settings=await api('/api/settings');if($('fCompany')){$('fCompany').value=settings.company_name||'';$('fCnpj').value=settings.cnpj||'';$('fIe').value=settings.ie||'';$('fPhone').value=settings.phone||'';$('fAddress').value=settings.address||'';$('fSeries').value=settings.nfc_series||'1';$('fCsc').value=settings.nfc_csc||''}}
async function saveFiscal(){let d=await api('/api/settings',{method:'POST',body:JSON.stringify({company_name:$('fCompany').value,cnpj:$('fCnpj').value,ie:$('fIe').value,phone:$('fPhone').value,address:$('fAddress').value,nfc_series:$('fSeries').value,nfc_csc:$('fCsc').value})});if(d.error)return alert(d.error);alert('Configurações salvas.');loadFiscal()}
function testPrint(){let w=window.open('','_blank','width=420,height=600');w.document.write('<body style="font-family:monospace;width:300px;margin:auto"><h2 style="text-align:center">CONVENIÊNCIA IRMÃO METRARIA</h2><p style="text-align:center">TESTE DE IMPRESSÃO</p><hr><p>Impressora configurada pelo navegador.</p><script>window.print();setTimeout(()=>window.close(),500)<\/script></body>');w.document.close()}
async function loadSettings(){let d=await api('/api/me');if($('myName')){$('myName').value=d.display_name;$('myUser').value=d.username}loadUsers()}
async function changeMine(){let d=await api('/api/me',{method:'POST',body:JSON.stringify({display_name:$('myName').value,username:$('myUser').value,old_password:$('oldPass').value,new_password:$('newPass').value})});if(d.error)return alert(d.error);alert('Seus dados foram alterados. Faça login novamente se o login mudou.');me=d.user;loadSettings()}
async function loadUsers(){let a=await api('/api/users');if($('usersTable'))$('usersTable').innerHTML='<table><tr><th>Nome</th><th>Login</th><th>Perfil</th><th></th></tr>'+a.map(x=>`<tr><td>${esc(x.display_name)}</td><td>${esc(x.username)}</td><td>${esc(x.role)}</td><td>${me.role==='admin'?`<button class="btn small blue" onclick="editUser(${x.id})">Editar</button>`:''}</td></tr>`).join('')+'</table>'}
async function editUser(id){let a=await api('/api/users');let u=a.find(x=>x.id===id);if(!u)return;let n=prompt('Nome exibido:',u.display_name);if(n===null)return;let un=prompt('Login:',u.username);if(un===null)return;let np=prompt('Nova senha (deixe vazio para não alterar):','');let d=await api('/api/users/'+id,{method:'POST',body:JSON.stringify({display_name:n,username:un,new_password:np})});if(d.error)return alert(d.error);loadUsers()}
async function downloadBackup(){let r=await fetch('/api/backup');let blob=await r.blob();let a=document.createElement('a');a.href=URL.createObjectURL(blob);a.download='backup_nosso_sonho_'+new Date().toISOString().slice(0,10)+'.db';a.click()}
function today(){return new Date().toISOString().slice(0,10)}
(async()=>{try{let d=await api('/api/me');if(d.id){me=d;renderApp()}else showLogin()}catch(e){showLogin()}})();
</script></body></html>'''

class H(BaseHTTPRequestHandler):
    def log_message(self, fmt, *args): pass
    def send_json(self,obj,status=200,headers=None):
        b=json.dumps(obj,ensure_ascii=False).encode();self.send_response(status);self.send_header('Content-Type','application/json; charset=utf-8');self.send_header('Content-Length',str(len(b)));(headers or {}).copy().items();
        for k,v in (headers or {}).items(): self.send_header(k,v)
        self.end_headers();self.wfile.write(b)
    def send_html(self,s):
        b=s.encode();self.send_response(200);self.send_header('Content-Type','text/html; charset=utf-8');self.send_header('Content-Length',str(len(b)));self.end_headers();self.wfile.write(b)
    def read_json(self):
        n=int(self.headers.get('Content-Length','0'));raw=self.rfile.read(n) or b'{}'
        try:return json.loads(raw)
        except:return {}
    def do_GET(self):
        p=urlparse(self.path)
        if p.path=='/': return self.send_html(HTML)
        if p.path=='/api/backup':
            u=require_user(self)
            if not u:return
            if not os.path.exists(DB):return self.send_json({'error':'Banco não encontrado.'},404)
            b=open(DB,'rb').read();self.send_response(200);self.send_header('Content-Type','application/octet-stream');self.send_header('Content-Disposition','attachment; filename="nosso_sonho.db"');self.send_header('Content-Length',str(len(b)));self.end_headers();self.wfile.write(b);return
        if p.path.startswith('/api/'): return self.api_get(p)
        fp=os.path.join(BASE,p.path.lstrip('/'))
        if os.path.isfile(fp):
            typ=mimetypes.guess_type(fp)[0] or 'application/octet-stream';b=open(fp,'rb').read();self.send_response(200);self.send_header('Content-Type',typ);self.send_header('Content-Length',str(len(b)));self.end_headers();self.wfile.write(b);return
        self.send_error(404)
    def do_POST(self):
        p=urlparse(self.path);data=self.read_json()
        if p.path=='/api/login':
            c=db();u=c.execute('SELECT * FROM users WHERE lower(username)=lower(?) AND active=1',(data.get('username',''),)).fetchone();c.close()
            if not u or u['password_hash']!=hash_pw(data.get('password','')):return self.send_json({'error':'Login ou senha inválidos.'},401)
            token=secrets.token_urlsafe(32);SESSIONS[token]={'id':u['id'],'username':u['username'],'display_name':u['display_name'],'role':u['role']}
            return self.send_json({'user':SESSIONS[token]},headers={'Set-Cookie':f'session={token}; Path=/; HttpOnly; SameSite=Lax'})
        if p.path=='/api/logout':
            tok=next((x.split('=',1)[1] for x in self.headers.get('Cookie','').split(';') if x.strip().startswith('session=')),None);SESSIONS.pop(tok,None);return self.send_json({'ok':True},headers={'Set-Cookie':'session=; Path=/; Max-Age=0'})
        u=require_user(self)
        if not u:return
        return self.api_post(p,data,u)
    def api_get(self,p):
        u=require_user(self)
        if not u:return
        c=db()
        try:
            if p.path=='/api/me':return self.send_json(dict(c.execute('SELECT id,username,display_name,role FROM users WHERE id=?',(u['id'],)).fetchone()))
            if p.path=='/api/users':return self.send_json([dict(x) for x in c.execute('SELECT id,username,display_name,role,active FROM users ORDER BY id').fetchall()])
            if p.path=='/api/settings':return self.send_json(settings(c))
            if p.path=='/api/products':return self.send_json([dict(x) for x in c.execute('SELECT * FROM products WHERE active=1 ORDER BY name').fetchall()])
            if p.path=='/api/customers':return self.send_json([dict(x) for x in c.execute('SELECT * FROM customers ORDER BY name').fetchall()])
            if p.path=='/api/suppliers':return self.send_json([dict(x) for x in c.execute('SELECT * FROM suppliers ORDER BY name').fetchall()])
            if p.path=='/api/expenses':return self.send_json([dict(x) for x in c.execute('SELECT * FROM expenses ORDER BY id DESC LIMIT 200').fetchall()])
            if p.path=='/api/cash/status':
                s=c.execute('SELECT * FROM cash_sessions WHERE closed_at IS NULL ORDER BY id DESC LIMIT 1').fetchone();
                if not s:return self.send_json({'open':False})
                expected=self.cash_expected(c,s['id'],s['opening_amount']);return self.send_json({'open':True,'id':s['id'],'opened_at':s['opened_at'],'opening_amount':s['opening_amount'],'expected':expected})
            if p.path=='/api/cash':
                s=c.execute('SELECT * FROM cash_sessions WHERE closed_at IS NULL ORDER BY id DESC LIMIT 1').fetchone();opening=s['opening_amount'] if s else 0;sid=s['id'] if s else None
                d=today();cash_sales=c.execute("SELECT COALESCE(SUM(total),0) FROM sales WHERE date(created_at)=? AND payment='dinheiro'",(d,)).fetchone()[0]
                entries=c.execute("SELECT COALESCE(SUM(amount),0) FROM cash_movements WHERE date(created_at)=? AND kind='entrada'",(d,)).fetchone()[0]
                exits=c.execute("SELECT COALESCE(SUM(amount),0) FROM cash_movements WHERE date(created_at)=? AND kind='saida'",(d,)).fetchone()[0]
                moves=c.execute('SELECT * FROM cash_movements WHERE date(created_at)=? ORDER BY id DESC',(d,)).fetchall();expected=opening+cash_sales+entries-exits
                return self.send_json({'opening':opening,'cashSales':cash_sales,'entries':entries,'exits':exits,'expected':expected,'movements':[dict(x) for x in moves]})
            if p.path=='/api/dashboard':return self.dashboard(c)
            if p.path=='/api/report':
                q=parse_qs(p.query);d=q.get('date',[today()])[0];tot,count=c.execute('SELECT COALESCE(SUM(total),0),COUNT(*) FROM sales WHERE date(created_at)=?',(d,)).fetchone();items=c.execute('SELECT COALESCE(SUM(qty),0) FROM sale_items si JOIN sales s ON s.id=si.sale_id WHERE date(s.created_at)=?',(d,)).fetchone()[0]
                rows=c.execute('''SELECT si.name,SUM(si.qty) qty,SUM(si.subtotal) total,SUM(si.qty*p.cost) cost FROM sale_items si JOIN sales s ON s.id=si.sale_id LEFT JOIN products p ON p.id=si.product_id WHERE date(s.created_at)=? GROUP BY si.name ORDER BY total DESC''',(d,)).fetchall();payments={r['payment']:r['total'] for r in c.execute('SELECT payment,SUM(total) total FROM sales WHERE date(created_at)=? GROUP BY payment',(d,)).fetchall()};profit=sum((r['total'] or 0)-(r['cost'] or 0) for r in rows);return self.send_json({'date':d,'total':tot,'count':count,'items':items,'profit':profit,'payments':payments,'products':[{'name':r['name'],'qty':r['qty'],'total':r['total'],'cost':r['cost'] or 0,'profit':(r['total'] or 0)-(r['cost'] or 0)} for r in rows]})
            return self.send_json({'error':'not found'},404)
        finally:c.close()
    def cash_expected(self,c,sid,opening):
        s=c.execute('SELECT opened_at,closed_at FROM cash_sessions WHERE id=?',(sid,)).fetchone();start=s['opened_at'];end=s['closed_at'] or '9999-12-31'
        cash_sales=c.execute("SELECT COALESCE(SUM(total),0) FROM sales WHERE payment='dinheiro' AND created_at>=? AND created_at<=?",(start,end)).fetchone()[0]
        ent=c.execute("SELECT COALESCE(SUM(amount),0) FROM cash_movements WHERE kind='entrada' AND created_at>=? AND created_at<=?",(start,end)).fetchone()[0]
        out=c.execute("SELECT COALESCE(SUM(amount),0) FROM cash_movements WHERE kind='saida' AND created_at>=? AND created_at<=?",(start,end)).fetchone()[0]
        return money(opening+cash_sales+ent-out)
    def dashboard(self,c):
        d=today();sales,count=c.execute('SELECT COALESCE(SUM(total),0),COUNT(*) FROM sales WHERE date(created_at)=?',(d,)).fetchone();items=c.execute('SELECT COALESCE(SUM(qty),0) FROM sale_items si JOIN sales s ON s.id=si.sale_id WHERE date(s.created_at)=?',(d,)).fetchone()[0];low=c.execute('SELECT COUNT(*) FROM products WHERE active=1 AND stock<=min_stock').fetchone()[0];profit=c.execute('SELECT COALESCE(SUM(si.subtotal-si.qty*COALESCE(p.cost,0)),0) FROM sale_items si JOIN sales s ON s.id=si.sale_id LEFT JOIN products p ON p.id=si.product_id WHERE date(s.created_at)=?',(d,)).fetchone()[0];exp=c.execute('SELECT COALESCE(SUM(amount),0) FROM expenses WHERE date(created_at)=? AND paid=1',(d,)).fetchone()[0];payments={r['payment']:r['total'] for r in c.execute('SELECT payment,SUM(total) total FROM sales WHERE date(created_at)=? GROUP BY payment',(d,)).fetchall()};return self.send_json({'sales':sales,'count':count,'items':items,'low':low,'profit':profit,'expenses':exp,'payments':payments})
    def api_post(self,p,data,u):
        c=db()
        try:
            if p.path=='/api/me':
                cur=c.execute('SELECT * FROM users WHERE id=?',(u['id'],)).fetchone();
                if not cur or cur['password_hash']!=hash_pw(data.get('old_password','')):return self.send_json({'error':'Senha atual incorreta.'},400)
                un=(data.get('username') or cur['username']).strip();dn=(data.get('display_name') or cur['display_name']).strip();
                if not un or not dn:return self.send_json({'error':'Nome e login são obrigatórios.'},400)
                try:c.execute('UPDATE users SET username=?,display_name=? WHERE id=?',(un,dn,u['id']))
                except sqlite3.IntegrityError:return self.send_json({'error':'Esse login já existe.'},400)
                np=data.get('new_password','');
                if np:c.execute('UPDATE users SET password_hash=? WHERE id=?',(hash_pw(np),u['id']))
                c.commit();nu={'id':u['id'],'username':un,'display_name':dn,'role':cur['role']};
                tok=next((x.split('=',1)[1] for x in self.headers.get('Cookie','').split(';') if x.strip().startswith('session=')),None);SESSIONS[tok]=nu;return self.send_json({'user':nu})
            if p.path.startswith('/api/users/'):
                if u['role']!='admin':return self.send_json({'error':'Somente administrador pode editar usuários.'},403)
                uid=int(p.path.rsplit('/',1)[1]);np=data.get('new_password','');
                try:
                    c.execute('UPDATE users SET username=?,display_name=? WHERE id=?',(data.get('username'),data.get('display_name'),uid));
                    if np:c.execute('UPDATE users SET password_hash=? WHERE id=?',(hash_pw(np),uid))
                    c.commit();return self.send_json({'ok':True})
                except sqlite3.IntegrityError:return self.send_json({'error':'Login já existe.'},400)
            if p.path=='/api/settings':
                for k,v in data.items():c.execute('INSERT INTO settings(k,v) VALUES(?,?) ON CONFLICT(k) DO UPDATE SET v=excluded.v',(k,str(v)))
                c.commit();return self.send_json({'ok':True})
            if p.path=='/api/products' or p.path.startswith('/api/products/'):
                pid=int(p.path.rsplit('/',1)[1]) if p.path.startswith('/api/products/') else None;img=data.get('image','')
                if img and img.startswith('data:'):
                    head,raw=img.split(',',1);ext=head.split(';')[0].split('/')[-1].replace('jpeg','jpg');name=f"{datetime.now().strftime('%Y%m%d%H%M%S%f')}.{ext}";open(os.path.join(UPLOADS,name),'wb').write(base64.b64decode(raw));img='/static/uploads/'+name
                if pid:
                    old=c.execute('SELECT stock FROM products WHERE id=?',(pid,)).fetchone();c.execute('UPDATE products SET name=?,barcode=?,category=?,cost=?,price=?,stock=?,min_stock=?,image=? WHERE id=?',(data.get('name'),data.get('barcode') or None,data.get('category'),money(data.get('cost')),money(data.get('price')),float(data.get('stock',0)),float(data.get('min_stock',0)),img,pid));diff=float(data.get('stock',0))-(old['stock'] if old else 0)
                    if abs(diff)>1e-9:c.execute('INSERT INTO stock_movements(product_id,kind,qty,reason,user_id,created_at) VALUES(?,?,?,?,?,?)',(pid,'ajuste',diff,'Edição do produto',u['id'],now()))
                else:
                    cur=c.execute('INSERT INTO products(name,barcode,category,cost,price,stock,min_stock,image,created_at) VALUES(?,?,?,?,?,?,?,?,?)',(data.get('name'),data.get('barcode') or None,data.get('category'),money(data.get('cost')),money(data.get('price')),float(data.get('stock',0)),float(data.get('min_stock',0)),img,now()));pid=cur.lastrowid
                    if float(data.get('stock',0)):c.execute('INSERT INTO stock_movements(product_id,kind,qty,reason,user_id,created_at) VALUES(?,?,?,?,?,?)',(pid,'entrada',float(data.get('stock',0)),'Cadastro inicial',u['id'],now()))
                c.commit();return self.send_json({'id':pid})
            if p.path=='/api/sales':
                sess=c.execute('SELECT * FROM cash_sessions WHERE closed_at IS NULL ORDER BY id DESC LIMIT 1').fetchone();
                if not sess:return self.send_json({'error':'Abra o caixa antes de vender.'},400)
                items=data.get('items',[]);total=money(sum(float(x['price'])*float(x['qty']) for x in items));payment=data.get('payment');received=money(data.get('received',total));change=money(max(0,received-total));
                for x in items:
                    pr=c.execute('SELECT stock FROM products WHERE id=? AND active=1',(x['id'],)).fetchone()
                    if not pr or pr['stock']<float(x['qty']):return self.send_json({'error':f"Estoque insuficiente para {x.get('name','produto')}"},400)
                if payment=='dinheiro' and received<total:return self.send_json({'error':'Valor recebido menor que o total.'},400)
                cur=c.execute('INSERT INTO sales(total,payment,received,change_amount,user_id,customer_id,created_at) VALUES(?,?,?,?,?,?,?)',(total,payment,received,change,u['id'],data.get('customer_id') or None,now()));sid=cur.lastrowid;out=[]
                for x in items:
                    sub=money(float(x['price'])*float(x['qty']));c.execute('INSERT INTO sale_items(sale_id,product_id,name,qty,unit_price,subtotal) VALUES(?,?,?,?,?,?)',(sid,x['id'],x['name'],x['qty'],x['price'],sub));c.execute('UPDATE products SET stock=stock-? WHERE id=?',(x['qty'],x['id']));c.execute('INSERT INTO stock_movements(product_id,kind,qty,reason,user_id,created_at) VALUES(?,?,?,?,?,?)',(x['id'],'saida',-float(x['qty']),'Venda #'+str(sid),u['id'],now()));out.append({'name':x['name'],'qty':x['qty'],'subtotal':sub})
                c.commit();return self.send_json({'sale':{'id':sid,'total':total,'payment':payment,'received':received,'change_amount':change,'created_at':now(),'items':out}})
            if p.path=='/api/cash/open':
                if c.execute('SELECT 1 FROM cash_sessions WHERE closed_at IS NULL').fetchone():return self.send_json({'error':'Já existe um caixa aberto.'},400)
                cur=c.execute('INSERT INTO cash_sessions(opened_by,opened_at,opening_amount) VALUES(?,?,?)',(u['id'],now(),money(data.get('amount'))));c.commit();return self.send_json({'id':cur.lastrowid})
            if p.path=='/api/cash/close':
                s=c.execute('SELECT * FROM cash_sessions WHERE closed_at IS NULL ORDER BY id DESC LIMIT 1').fetchone();
                if not s:return self.send_json({'error':'Nenhum caixa aberto.'},400)
                expected=self.cash_expected(c,s['id'],s['opening_amount']);counted=money(data.get('amount'));diff=money(counted-expected);closed=now();c.execute('UPDATE cash_sessions SET closed_at=?,closing_amount=?,expected_amount=?,difference=? WHERE id=?',(closed,counted,expected,diff,s['id']));c.commit();return self.send_json({'expected':expected,'counted':counted,'difference':diff})
            if p.path=='/api/cash':
                amount=money(data.get('amount'));kind=data.get('kind');
                if amount<=0:return self.send_json({'error':'Valor inválido.'},400)
                c.execute('INSERT INTO cash_movements(kind,amount,description,user_id,created_at) VALUES(?,?,?,?,?)',(kind,amount,data.get('description'),u['id'],now()));c.commit();return self.send_json({'ok':True})
            if p.path=='/api/stock':
                pid=int(data.get('product_id'));qty=float(data.get('qty') or 0);kind=data.get('kind');p0=c.execute('SELECT stock FROM products WHERE id=?',(pid,)).fetchone();
                if not p0 or qty<=0:return self.send_json({'error':'Produto/quantidade inválidos.'},400)
                if kind=='entrada':delta=qty
                elif kind=='perda':delta=-qty
                else:delta=qty-float(p0['stock'])
                new=max(0,float(p0['stock'])+delta);c.execute('UPDATE products SET stock=? WHERE id=?',(new,pid));c.execute('INSERT INTO stock_movements(product_id,kind,qty,reason,user_id,created_at) VALUES(?,?,?,?,?,?)',(pid,kind,delta,data.get('reason'),u['id'],now()));c.commit();return self.send_json({'stock':new})
            if p.path=='/api/customers':
                c.execute('INSERT INTO customers(name,phone,cpf_cnpj,address,created_at) VALUES(?,?,?,?,?)',(data.get('name'),data.get('phone'),data.get('cpf_cnpj'),data.get('address'),now()));c.commit();return self.send_json({'ok':True})
            if p.path=='/api/suppliers':
                c.execute('INSERT INTO suppliers(name,phone,cpf_cnpj,notes,created_at) VALUES(?,?,?,?,?)',(data.get('name'),data.get('phone'),data.get('cpf_cnpj'),data.get('notes'),now()));c.commit();return self.send_json({'ok':True})
            if p.path=='/api/expenses':
                amount=money(data.get('amount'));c.execute('INSERT INTO expenses(description,category,amount,due_date,paid,created_at,user_id) VALUES(?,?,?,?,?,?,?)',(data.get('description'),data.get('category'),amount,data.get('due_date'),1 if data.get('paid') else 0,now(),u['id']));c.commit();return self.send_json({'ok':True})
            return self.send_json({'error':'not found'},404)
        except sqlite3.IntegrityError as e:
            c.rollback();return self.send_json({'error':'Dados duplicados ou inválidos. '+str(e)},400)
        except Exception as e:
            c.rollback();return self.send_json({'error':'Erro: '+str(e)},500)
        finally:c.close()

init_db()
if __name__=='__main__':
    port=8080
    ip='127.0.0.1'
    try: ip=socket.gethostbyname(socket.gethostname())
    except: pass
    print('='*55);print(APP_NAME);print('PC:      http://localhost:%d'%port);print('CELULAR: http://%s:%d'%(ip,port));print('Se o celular não abrir, deixe o PC e celular na mesma Wi-Fi e libere Python no Firewall do Windows.');print('='*55)
    ThreadingHTTPServer(('0.0.0.0',port),H).serve_forever()
