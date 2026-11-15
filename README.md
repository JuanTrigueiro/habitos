<!DOCTYPE html>
<html lang="pt-BR">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1, viewport-fit=cover">
<meta name="apple-mobile-web-app-capable" content="yes">
<meta name="apple-mobile-web-app-title" content="Hábitos">
<meta name="theme-color" content="#0f172a">
<title>Hábitos</title>
<style>
  :root { box-sizing: border-box; padding-top: env(safe-area-inset-top, 0px); padding-bottom: env(safe-area-inset-bottom, 0px); }
  *, *::before, *::after { box-sizing: inherit; }
  body { margin: 0; font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, sans-serif; background: #0f172a; color: #e2e8f0; min-height: 100vh; }
  main { max-width: 480px; margin: 0 auto; padding: 16px; }
  h1 { font-size: 22px; margin: 8px 0 12px; }
  form { display: flex; gap: 8px; margin-bottom: 16px; }
  input { flex: 1; padding: 12px; border-radius: 10px; border: 1px solid #334155; background: #1e293b; color: #e2e8f0; font-size: 16px; min-width: 0; }
  button { border: none; border-radius: 10px; padding: 10px 14px; font-size: 16px; cursor: pointer; background: #334155; color: #e2e8f0; }
  button:disabled { opacity: 0.3; cursor: default; }
  .add { background: #22c55e; color: #052e16; font-weight: 600; }
  .semana { display: flex; align-items: center; justify-content: space-between; background: #1e293b; border-radius: 12px; padding: 8px; margin-bottom: 12px; }
  .semana span { font-size: 15px; font-weight: 600; }
  .semana button { padding: 6px 12px; }
  .habit { background: #1e293b; border-radius: 12px; padding: 12px; margin-bottom: 10px; }
  .topo { display: flex; align-items: center; gap: 8px; }
  .info { flex: 1; min-width: 0; }
  .name { font-weight: 600; overflow: hidden; text-overflow: ellipsis; white-space: nowrap; }
  .stats { font-size: 13px; color: #94a3b8; margin-top: 2px; }
  .del { background: transparent; color: #64748b; padding: 4px 8px; font-size: 18px; }
  .dias { display: grid; grid-template-columns: repeat(7, 1fr); gap: 4px; margin-top: 10px; }
  .dia { display: flex; flex-direction: column; align-items: center; padding: 6px 0; font-size: 12px; background: #334155; }
  .dia small { font-size: 11px; opacity: 0.8; }
  .dia.feito { background: #22c55e; color: #052e16; font-weight: 600; }
  .dia.hoje { outline: 2px solid #38bdf8; outline-offset: -2px; }
  .cabecalho { display: grid; grid-template-columns: repeat(7, 1fr); gap: 4px; font-size: 11px; color: #64748b; text-align: center; margin-bottom: 4px; }
  .empty, .aviso { color: #64748b; text-align: center; margin-top: 40px; }
  .erro { color: #f87171; text-align: center; margin-top: 20px; font-size: 14px; }
</style>
</head>
<body>
<main>
  <h1>Meus hábitos</h1>
  <form id="form">
    <input id="novo" placeholder="Novo hábito (ex: ler 10 min)" autocomplete="off" maxlength="60">
    <button class="add" type="submit">Adicionar</button>
  </form>
  <div class="semana">
    <button id="anterior" type="button">‹</button>
    <span id="titulo"></span>
    <button id="proxima" type="button">›</button>
  </div>
  <div class="cabecalho"><div>D</div><div>S</div><div>T</div><div>Q</div><div>Q</div><div>S</div><div>S</div></div>
  <div id="lista"><p class="aviso">Carregando...</p></div>
  <div id="erro" class="erro"></div>
</main>

<script src="https://cdn.jsdelivr.net/npm/@supabase/supabase-js@2.45.4/dist/umd/supabase.min.js"></script>
<script>
  const SUPABASE_URL = "https://yfbcvttsrlfboloavolv.supabase.co";
  const SUPABASE_KEY = "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJzdXBhYmFzZSIsInJlZiI6InlmYmN2dHRzcmxmYm9sb2F2b2x2Iiwicm9sZSI6ImFub24iLCJpYXQiOjE3OTQ3MjI4NzYsImV4cCI6MjExMDI5ODg3Nn0.LLzh8lfuZVJ5biGppA2u_YY6wgmKbOkXIqZf41YzNPQ";
  const sb = supabase.createClient(SUPABASE_URL, SUPABASE_KEY);

  const NOMES_DIAS = ["D", "S", "T", "Q", "Q", "S", "S"];
  const MESES = ["jan", "fev", "mar", "abr", "mai", "jun", "jul", "ago", "set", "out", "nov", "dez"];

  let estado = [];          // [{id, nome, dias: ["YYYY-MM-DD", ...]}]
  let deslocamento = 0;     // 0 = semana atual

  const p2 = n => String(n).padStart(2, "0");
  function chaveData(d) { return d.getFullYear() + "-" + p2(d.getMonth() + 1) + "-" + p2(d.getDate()); }
  function hojeKey() { return chaveData(new Date()); }

  function domingoDaSemana() {
    const h = new Date();
    return new Date(h.getFullYear(), h.getMonth(), h.getDate() - h.getDay() + deslocamento * 7);
  }

  function sequencia(h) {
    const set = new Set(h.dias);
    const d = new Date();
    if (!set.has(chaveData(d))) d.setDate(d.getDate() - 1);
    let n = 0;
    while (set.has(chaveData(d))) { n++; d.setDate(d.getDate() - 1); }
    return n;
  }

  function mostrarErro(msg) { document.getElementById("erro").textContent = msg || ""; }

  async function carregar() {
    const [h, r] = await Promise.all([
      sb.from("habitos").select("id, nome").order("criado_em"),
      sb.from("registros").select("habito_id, dia")
    ]);
    if (h.error || r.error) throw (h.error || r.error);
    estado = h.data.map(x => ({
      id: x.id,
      nome: x.nome,
      dias: r.data.filter(y => y.habito_id === x.id).map(y => y.dia)
    }));
  }

  async function alternarDia(id, k) {
    const h = estado.find(x => x.id === id);
    if (!h || k > hojeKey()) return;
    const marcado = h.dias.includes(k);
    h.dias = marcado ? h.dias.filter(x => x !== k) : [...h.dias, k];
    render();
    const res = marcado
      ? await sb.from("registros").delete().eq("habito_id", id).eq("dia", k)
      : await sb.from("registros").insert({ habito_id: id, dia: k });
    if (res.error) mostrarErro("Não consegui salvar. Recarregue a página.");
  }

  async function remover(id) {
    const h = estado.find(x => x.id === id);
    if (!h || !confirm('Remover "' + h.nome + '"?')) return;
    const res = await sb.from("habitos").delete().eq("id", id);
    if (res.error) return mostrarErro("Não consegui remover.");
    estado = estado.filter(x => x.id !== id);
    render();
  }

  function render() {
    const dom = domingoDaSemana();
    const sab = new Date(dom.getFullYear(), dom.getMonth(), dom.getDate() + 6);
    document.getElementById("titulo").textContent =
      dom.getDate() + " " + MESES[dom.getMonth()] + " – " + sab.getDate() + " " + MESES[sab.getMonth()];
    document.getElementById("proxima").disabled = deslocamento >= 0;

    const lista = document.getElementById("lista");
    lista.innerHTML = "";
    if (estado.length === 0) {
      lista.innerHTML = '<p class="empty">Nenhum hábito ainda. Adicione o primeiro acima.</p>';
      return;
    }

    const hoje = hojeKey();
    estado.forEach(h => {
      const card = document.createElement("div");
      card.className = "habit";

      const topo = document.createElement("div");
      topo.className = "topo";
      const info = document.createElement("div");
      info.className = "info";
      const nome = document.createElement("div");
      nome.className = "name";
      nome.textContent = h.nome;
      const stats = document.createElement("div");
      stats.className = "stats";
      stats.textContent = "🔥 " + sequencia(h) + " dia(s) seguidos · " + h.dias.length + " no total";
      info.append(nome, stats);
      const del = document.createElement("button");
      del.className = "del";
      del.textContent = "×";
      del.setAttribute("aria-label", "Remover hábito");
      del.onclick = () => remover(h.id);
      topo.append(info, del);

      const grade = document.createElement("div");
      grade.className = "dias";
      for (let i = 0; i < 7; i++) {
        const d = new Date(dom.getFullYear(), dom.getMonth(), dom.getDate() + i);
        const k = chaveData(d);
        const b = document.createElement("button");
        b.className = "dia" + (h.dias.includes(k) ? " feito" : "") + (k === hoje ? " hoje" : "");
        b.disabled = k > hoje;
        b.innerHTML = "<span>" + NOMES_DIAS[i] + "</span><small>" + d.getDate() + "</small>";
        b.setAttribute("aria-label", d.getDate() + " de " + MESES[d.getMonth()]);
        b.onclick = () => alternarDia(h.id, k);
        grade.append(b);
      }

      card.append(topo, grade);
      lista.append(card);
    });
  }

  document.getElementById("form").addEventListener("submit", async e => {
    e.preventDefault();
    const input = document.getElementById("novo");
    const nome = input.value.trim();
    if (!nome) return;
    const res = await sb.from("habitos").insert({ nome }).select("id, nome").single();
    if (res.error) return mostrarErro("Não consegui adicionar.");
    estado.push({ id: res.data.id, nome: res.data.nome, dias: [] });
    input.value = "";
    mostrarErro("");
    render();
  });
  document.getElementById("anterior").onclick = () => { deslocamento--; render(); };
  document.getElementById("proxima").onclick = () => { if (deslocamento < 0) { deslocamento++; render(); } };

  carregar().then(render).catch(() => {
    document.getElementById("lista").innerHTML = "";
    mostrarErro("Não consegui carregar os dados do Supabase.");
  });
</script>
</body>
</html>
