<!-- NOVO CÓDIGO OTIMIZADO E FUNCIONAL -->
<!DOCTYPE html>
<html lang="pt-BR">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>Controle de Cortes</title>
<style>
 body{background:#000;color:#e0e0e0;font-family:Arial;padding:15px;margin:0}
 h2,h3{text-align:center;color:#4da3ff}
 .card{background:#0f0f0f;padding:15px;border-radius:12px;border:1px solid #003366;margin-bottom:15px}
 input,button{width:100%;padding:12px;margin-top:8px;border-radius:8px;border:none;font-size:15px}
 input{background:#111;border:1px solid #222;color:white}
 button{background:#004f99;color:white}
 button:hover{background:#003f77}
 .btn-confirmar{background:#0f9d58!important}
 .cliente{background:#111;padding:12px;border-radius:10px;margin-top:10px;border-left:5px solid #1e90ff}
 .progressBar{background:#333;height:12px;border-radius:8px;margin-top:5px}
 .progress{background:#1e90ff;height:12px;border-radius:8px;width:0%}
</style>
</head>
<body>
<h2>📋 Controle de Cortes</h2>

<div class="card">
 <h3>Adicionar Cliente</h3>
 <input id="nome" placeholder="Nome do cliente">
 <input id="telefone" placeholder="Telefone (números)">
 <input id="valorPlano" type="number" placeholder="Valor do plano (R$)">
 <input id="totalCortes" type="number" placeholder="Total de cortes contratados">
 <button onclick="addCliente()">Adicionar Cliente</button>
</div>

<div class="card">
 <h3>Buscar Cliente</h3>
 <input id="busca" placeholder="Buscar por nome" oninput="render()">
</div>

<div id="lista"></div>

<div class="card">
 <button onclick="exportar()">📄 Exportar TXT</button>
 <button onclick="enviarWhats()">📩 Enviar WhatsApp</button>
</div>

<div class="card">
 <h3>Dashboard Geral</h3>
 <div id="dash"></div>
</div>

<script>
let clientes = JSON.parse(localStorage.getItem("clientes") || "[]");
function save(){localStorage.setItem("clientes",JSON.stringify(clientes));}

function addCliente(){
 let nome = document.getElementById("nome").value.trim();
 let tel = document.getElementById("telefone").value.replace(/\D/g,"");
 let valor = Number(document.getElementById("valorPlano").value);
 let total = Number(document.getElementById("totalCortes").value);
 if(!nome||!valor||!total){alert("Preencha tudo!");return;}
 clientes.push({nome,telefone:tel,valor,total,feitos:0,hist:[]});
 save(); render();
 document.getElementById("nome").value="";
 document.getElementById("telefone").value="";
 document.getElementById("valorPlano").value="";
 document.getElementById("totalCortes").value="";
}

function corte(i){
 let c = clientes[i];
 if(c.feitos>=c.total){alert("Plano já finalizado!");return;}
 c.feitos++;
 c.hist.push(new Date().toLocaleString());
 if(c.feitos===c.total && c.telefone){
  let msg = `Olá ${c.nome}, seu plano foi concluído!`;
  window.open(`https://wa.me/${c.telefone}?text=${encodeURIComponent(msg)}`);
 }
 save(); render();
}

function pagar(i){alert(`Pagamento confirmado para ${clientes[i].nome}`);}

function hist(i){alert(clientes[i].hist.join("\n")||"Sem cortes ainda");}

function render(){
 let b = document.getElementById("busca").value.toLowerCase();
 lista.innerHTML="";
 clientes.filter(c=>c.nome.toLowerCase().includes(b)).forEach((c,i)=>{
  let pct = Math.round((c.feitos/c.total)*100);
  lista.innerHTML+=`
   <div class='cliente'>
    <b>${c.nome}</b><br>
    Plano: R$ ${c.valor.toFixed(2)}<br>
    ${c.feitos}/${c.total}
    <div class='progressBar'><div class='progress' style='width:${pct}%'></div></div>
    <button onclick='corte(${i})'>Registrar Corte</button>
    <button class='btn-confirmar' onclick='pagar(${i})'>Confirmar Pagamento</button>
    <button onclick='hist(${i})'>Histórico</button>
   </div>`;
 });
 dashboard();
}

function dashboard(){
 let total = clientes.length;
 let tCortes = clientes.reduce((s,c)=>s+c.total,0);
 let tFeitos = clientes.reduce((s,c)=>s+c.feitos,0);
 dash.innerHTML = `Clientes: <b>${total}</b><br>Cortes: <b>${tFeitos}/${tCortes}</b>`;
}

function exportar(){
 let blob = new Blob([JSON.stringify(clientes,null,2)],{type:"text/plain"});
 let a=document.createElement("a");
 a.href=URL.createObjectURL(blob);
 a.download="relatorio.txt";
 a.click();
}

function enviarWhats(){
 let msg = "RELATÓRIO DE CORTES:%0A%0A";
 clientes.forEach(c=> msg += `${c.nome}: ${c.feitos}/${c.total}%0A`);
 window.open("https://wa.me/?text="+msg);
}

render();
</script>
</body>
</html>
