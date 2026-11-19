---
budget: 90000
currency: ₽
reserve: 17000
reserve_transferred: false
travel: 15000
travel_transferred: false
categories:
  Жилье: 6000
  Продукты: 20000
  Кафе: 7000
  Подписки и связь: 3000
  Спортзал: 3000
  Транспорт / бензин: 3000
  Вещи для дома: 2000
  Развлечения: 3000
  Подарки: 3000
  Здоровье: 1500
  Одежда: 2000
  Техника / личные покупки: 2000
  Прочее: 2500
---

```dataviewjs
const { Notice, getFrontMatterInfo, parseYaml } = require("obsidian");
const host = dv.container;
const run = host.__financeRun = Symbol();

// Сохраняем высоту во время перерисовки, чтобы интерфейс не прыгал.
if (!host.__financeLayout) {
  const observer = new ResizeObserver(entries => {
    for (const { target } of entries) if (target.parentElement === host) host.style.minHeight = `${target.getBoundingClientRect().height}px`;
  });
  host.__financeLayout = observer;
  dv.component.register(() => {
    observer.disconnect();
    delete host.__financeLayout;
    host.style.removeProperty("min-height");
  });
}
host.__financeLayout.disconnect();

async function main() {
  const p = dv.current();
  const period = p.file.name;
  const match = /^(\d{4})-(0[1-9]|1[0-2])$/.exec(period);
  if (!match) {
    host.style.removeProperty("min-height");
    dv.paragraph("Название заметки должно быть YYYY-MM.");
    return;
  }

  const year = Number(match[1]);
  const month = Number(match[2]);
  const path = p.file.path;
  const folder = p.file.folder ? `${p.file.folder}/` : "";
  const file = app.vault.getAbstractFileByPath(path);
  if (!file) return;

  const EXPENSES = "📥 Расходы месяца";
  const FOOD = "🛒 Продукты по неделям";
  const TYPES = {
    "Жилье": "required", "Продукты": "required", "Подписки и связь": "required", "Спортзал": "required",
    "Кафе": "flex", "Транспорт / бензин": "flex", "Развлечения": "flex", "Прочее": "flex",
    "Вещи для дома": "fund", "Подарки": "fund", "Здоровье": "fund", "Одежда": "fund", "Техника / личные покупки": "fund"
  };
  // Тип fund означает переносимый остаток. Список определяется здесь, не в Properties.
  const carryNames = new Set(Object.keys(TYPES).filter(name => TYPES[name] === "fund"));
  const ICONS = {
    "Жилье": "🏠", "Продукты": "🛒", "Кафе": "🍔", "Подписки и связь": "📱", "Спортзал": "🏋️",
    "Транспорт / бензин": "🚗", "Вещи для дома": "🛋️", "Развлечения": "🎮", "Подарки": "🎁",
    "Здоровье": "💊", "Одежда": "👕", "Техника / личные покупки": "💻", "Прочее": "💳"
  };
  const isMonthly = name => name !== "Продукты" && name !== "Накопления";
  const iso = d => d.toISOString().slice(0, 10);
  const addDays = (d, n) => new Date(d.getTime() + n * 86400000);
  const monday = d => addDays(d, -((d.getUTCDay() + 6) % 7));
  const monthOf = d => iso(d).slice(0, 7);
  const now = new Date();
  const today = new Date(Date.UTC(now.getFullYear(), now.getMonth(), now.getDate()));
  const currentKey = iso(monday(today));
  const dateFormat = new Intl.DateTimeFormat("ru-RU", { day: "numeric", month: "short", timeZone: "UTC" });
  const dates = start => `${dateFormat.format(start)} – ${dateFormat.format(addDays(start, 6))}`;

  function weeksOf(per) {
    const [y, m] = per.split("-").map(Number);
    const last = new Date(Date.UTC(y, m, 0));
    const result = [];
    for (let d = monday(new Date(Date.UTC(y, m - 1, 1))); d <= last; d = addDays(d, 7)) result.push(d);
    return result;
  }

  function amount(value) {
    const text = String(value ?? "").replace(/[\s₽]/g, "").replace(",", ".");
    const n = Number(text);
    return /^\d+(\.\d+)?$/.test(text) && Number.isFinite(n) ? n : null;
  }

  function settings(source) {
    const info = getFrontMatterInfo(source);
    return info.exists ? parseYaml(info.frontmatter) || {} : {};
  }

  const limitsOf = data => Object.fromEntries(Object.entries(data.categories || {}).map(([name, value]) => [name, amount(value) ?? 0]));

  function viewsOf(filePath) {
    return app.workspace.getLeavesOfType("markdown").map(leaf => leaf.view).filter(view => view.file?.path === filePath);
  }

  function editorView() {
    const views = viewsOf(path).filter(view => view.getMode() === "source");
    return views.find(view => view.editor.hasFocus()) || views.find(view => view.contentEl.contains(host)) || views[0];
  }

  function editingProperties() {
    return viewsOf(path).some(view => {
      const focused = view.contentEl?.ownerDocument.activeElement;
      return focused && view.contentEl.contains(focused) && focused.matches("input, textarea, [contenteditable='true']");
    });
  }

  async function readSource(target) {
    const view = viewsOf(target.path).find(view => view.getMode() === "source");
    return view ? view.editor.getValue() : app.vault.read(target);
  }

  function table(source, heading) {
    const eol = source.includes("\r\n") ? "\r\n" : "\n";
    const lines = source.split(eol);
    const at = lines.findIndex(line => line.trim() === `## ${heading}`);
    if (at < 0) return null;
    let start = at + 1;
    while (start < lines.length && !/^#{1,6}\s/.test(lines[start].trim()) && !/^\|.*\|$/.test(lines[start].trim())) start++;
    if (start === lines.length || !/^\|.*\|$/.test(lines[start].trim())) start = at + 1;
    let end = start;
    while (end < lines.length && /^\|.*\|$/.test(lines[end].trim())) end++;
    const rows = lines.slice(start + 2, end).map(line => line.trim().slice(1, -1).split(/(?<!\\)\|/).map(cell => cell.trim().replace(/\\\|/g, "|")));
    return { lines, eol, start, end, rows };
  }

  function readMonthly(source) {
    return new Map((table(source, EXPENSES)?.rows || []).filter(([name]) => name && isMonthly(name)));
  }

  function readFood(source, per) {
    const weeks = weeksOf(per);
    const values = new Map();
    for (const [label, value] of table(source, FOOD)?.rows || []) {
      const found = /^([1-6])\s*·/.exec(label);
      const start = found && weeks[Number(found[1]) - 1];
      if (start) values.set(iso(start), value);
    }
    return values;
  }

  // Точечная синхронизация: значения расходов не затрагиваются, курсор не сбрасываем.
  function tableChanges(source, heading, header, labels, keyOf, refreshLabels = false) {
    const found = table(source, heading);
    if (!found) return [];
    const { lines, eol, start, end, rows } = found;
    if (end < lines.length && lines[end].trim().startsWith("|")) return [];
    const existing = new Map(rows.map(([label], i) => [keyOf(label), { label, line: start + 2 + i }]));
    const changes = [];
    const additions = [];
    for (const label of labels) {
      const key = keyOf(label);
      const saved = existing.get(key);
      const escaped = label.replace(/\|/g, "\\|");
      if (!saved) {
        additions.push(`| ${escaped} |  |`);
      } else if (refreshLabels && saved.label !== label) {
        const line = lines[saved.line];
        const from = line.indexOf("|") + 1;
        const to = line.indexOf("|", from);
        const cell = line.slice(from, to);
        changes.push({
          from: { line: saved.line, ch: from + cell.length - cell.trimStart().length },
          to: { line: saved.line, ch: to - cell.length + cell.trimEnd().length },
          text: escaped
        });
      }
    }
    if (start === end) additions.unshift(header, "|---|---:|");
    if (additions.length) {
      const from = end < lines.length ? { line: end, ch: 0 } : { line: lines.length - 1, ch: lines.at(-1).length };
      changes.push({ from, text: end < lines.length ? additions.join(eol) + eol : eol + additions.join(eol) });
    }
    return changes;
  }

  function changesFor(source) {
    const data = settings(source);
    const names = Object.keys(limitsOf(data)).filter(isMonthly);
    const unit = String(data.currency || "₽").replace(/\|/g, "\\|");
    const labels = weeksOf(period).map((start, i) => `${i + 1} · ${iso(start) === currentKey ? `**🟢 ${dates(start)} · текущая**` : dates(start)}`);
    return [
      ...tableChanges(source, EXPENSES, `| Категория | Потрачено, ${unit} |`, names, name => name),
      ...tableChanges(source, FOOD, `| Неделя (Пн–Вс) | Потрачено, ${unit} |`, labels, label => /^([1-6])\s*·/.exec(label)?.[1], true)
    ];
  }

  function synchronize(source) {
    const lines = source.split(/\r?\n/);
    const eol = source.includes("\r\n") ? "\r\n" : "\n";
    const offset = pos => lines.slice(0, pos.line).reduce((n, line) => n + line.length + eol.length, 0) + pos.ch;
    const changes = changesFor(source).map(change => ({ from: offset(change.from), to: offset(change.to || change.from), text: change.text }));
    for (const change of changes.sort((a, b) => b.from - a.from)) source = source.slice(0, change.from) + change.text + source.slice(change.to);
    return source;
  }

  let source = await readSource(file);
  if (host.__financeRun !== run) return;
  const view = editorView();
  const shown = viewsOf(path).some(v => v.getMode() === "preview" && v.contentEl.contains(host));
  try {
    if (view && !editingProperties()) {
      source = view.editor.getValue();
      const changes = changesFor(source);
      if (changes.length) view.editor.transaction({ changes });
      source = view.editor.getValue();
    } else if (!view && shown && !editingProperties() && synchronize(source) !== source) {
      source = await app.vault.process(file, latest => host.__financeRun === run && !editorView() && !editingProperties() ? synchronize(latest) : latest);
    }
  } catch (error) {
    console.error("Finance tables:", error);
    new Notice("Не удалось обновить финансовые таблицы.");
  }

  const data = settings(source);
  const limits = limitsOf(data);
  const income = amount(data.budget) ?? 0;
  const reserve = amount(data.reserve) ?? 0;
  const travel = amount(data.travel) ?? 0;
  const currency = String(data.currency || "₽");
  const money = n => `${Math.round(n).toLocaleString("ru-RU")} ${currency}`;
  const foodLimit = limits["Продукты"] || 0;
  // Начало новой системы накопительных конвертов. Старые заметки не влияют на остатки.
  const carryStart = "2026-11";
  const sum = values => [...values].reduce((total, raw) => total + (amount(raw) ?? 0), 0);

  // Остатки не записываем в YAML: их можно пересчитать из истории без редактирования заметок.
  const docs = new Map();
  const candidates = app.vault.getMarkdownFiles().filter(f => f.path.startsWith(folder) && f.path.slice(folder.length).match(/^\d{4}-(0[1-9]|1[0-2])\.md$/) && f.basename >= carryStart && f.basename <= period);
  await Promise.all(candidates.map(async f => {
    const text = f.path === path ? source : await readSource(f);
    docs.set(f.basename, { data: settings(text), costs: readMonthly(text), food: readFood(text, f.basename) });
  }));
  docs.set(period, { data, costs: readMonthly(source), food: readFood(source, period) });
  // Для недель, пересекающих месяцы, учитываем обе заметки, если они существуют.
  for (const offset of [-1, 1]) {
    const adjacent = monthOf(new Date(Date.UTC(year, month - 1 + offset, 1)));
    if (docs.has(adjacent)) continue;
    const other = app.vault.getAbstractFileByPath(`${folder}${adjacent}.md`);
    if (other?.extension === "md") {
      const text = await readSource(other);
      docs.set(adjacent, { data: settings(text), costs: readMonthly(text), food: readFood(text, adjacent) });
    }
  }
  if (host.__financeRun !== run) return;

  const records = [...docs.keys()].filter(k => k >= carryStart && k <= period).sort();
  const previous = {};
  for (const name of carryNames) previous[name] = 0;
  for (const per of records.filter(k => k < period)) {
    const doc = docs.get(per);
    const oldLimits = limitsOf(doc.data);
    for (const name of carryNames) {
      const spent = name === "Продукты" ? sum(doc.food.values()) : amount(doc.costs.get(name)) ?? 0;
      previous[name] += (oldLimits[name] || 0) - spent;
    }
  }

  const costs = docs.get(period).costs;
  const food = docs.get(period).food;
  const foodSpent = sum(food.values());
  const otherSpent = sum(costs.values());
  const spent = foodSpent + otherSpent;
  const spendingPlan = Object.values(limits).reduce((n, v) => n + v, 0);
  const allocated = spendingPlan + reserve + travel;
  const free = income - allocated;
  const thisMonthRemaining = income - reserve - travel - spent;
  const carryIn = Object.values(previous).reduce((n, v) => n + v, 0);
  const balances = Object.fromEntries(Object.keys(limits).map(name => {
    const used = name === "Продукты" ? foodSpent : amount(costs.get(name)) ?? 0;
    return [name, (previous[name] || 0) + limits[name] - used];
  }));

  // Перенос между неделями только внутри текущего месяца.
  // Каждую календарную неделю считаем по дням, принадлежащим этой заметке.
  // Так пограничная неделя не переносит деньги из одного месяца в другой.
  let weeklyCarry = 0;
  const weeks = weeksOf(period).map(start => {
    const key = iso(start);
    let base = 0;
    for (let i = 0; i < 7; i++) {
      const day = addDays(start, i);
      if (monthOf(day) === period) base += foodLimit / new Date(Date.UTC(year, month, 0)).getUTCDate();
    }
    const used = amount(food.get(key)) ?? 0;
    const visible = start <= today;
    const carry = visible ? weeklyCarry : 0;
    const available = base + carry;
    const leftover = available - used;
    if (visible) weeklyCarry = leftover;
    return { start, spent: used, limit: base, carry, available, leftover, current: key === currentKey, visible };
  });

  const root = dv.el("div", "");
  root.className = "finance-dashboard";
  const style = document.createElement("style");
  style.textContent = `
    .finance-dashboard {
      background: var(--background-primary);
      border: 1px solid var(--background-modifier-border);
      border-radius: 20px;
      padding: 20px;
      color: var(--text-normal);
      font-size: 14px;
      box-sizing: border-box;
    }

    .finance-dashboard * {
      box-sizing: border-box;
    }

    .finance-dashboard .header {
      display: flex;
      justify-content: space-between;
      align-items: center;
      gap: 12px;
      margin-bottom: 20px;
    }

    .finance-dashboard .title {
      font-size: 25px;
      font-weight: 750;
      line-height: 1.3;
    }

    .finance-dashboard .muted {
      color: var(--text-muted);
      font-size: 12px;
    }

    .finance-dashboard .stats {
      display: grid;
      grid-template-columns: repeat(2,minmax(0,1fr));
      gap: 10px;
      margin-bottom: 16px;
    }

    .finance-dashboard .stat {
      background: var(--background-secondary);
      border-radius: 12px;
      padding: 14px;
      min-width: 0;
    }

    .finance-dashboard .number {
      font-size: 25px;
      font-weight: 700;
      margin-top: 6px;
      overflow-wrap: anywhere;
    }

    .finance-dashboard .green {
      color: #28b782;
    }

    .finance-dashboard .red {
      color: #e16a6a;
    }

    .finance-dashboard .section {
      border-top: 1px solid var(--background-modifier-border);
      margin-top: 22px;
      padding-top: 20px;
    }

    .finance-dashboard .heading {
      font-size: 15px;
      font-weight: 700;
      margin-bottom: 15px;
    }

    .finance-dashboard .row {
      display: flex;
      justify-content: space-between;
      align-items: baseline;
      gap: 12px;
      margin-bottom: 8px;
    }

    .finance-dashboard .row > :last-child {
      text-align: right;
    }

    .finance-dashboard .progress {
      margin: 14px 0;
    }

    .finance-dashboard .track {
      height: 10px;
      border-radius: 20px;
      background: var(--background-modifier-border);
      overflow: hidden;
    }

    .finance-dashboard .fill {
      height: 100%;
      border-radius: 20px;
      background: #16a085;
    }

    .finance-dashboard .savings {
      border: 1px solid var(--background-modifier-border);
      border-radius: 12px;
      padding: 14px;
      margin: 16px 0;
    }

    .finance-dashboard .week-grid {
      display: grid;
      grid-template-columns: repeat(2,minmax(0,1fr));
      gap: 10px;
    }

    .finance-dashboard .week-card {
      padding: 13px;
      border-radius: 12px;
      border: 1px solid var(--background-modifier-border);
      min-width: 0;
    }

    .finance-dashboard .current {
      border-color: #16a085;
      background: rgba(22,160,133,.08);
    }

    .finance-dashboard .active {
      padding: 15px;
      border-radius: 14px;
      background: var(--background-secondary);
    }

    .finance-dashboard .week-title {
      font-size: 13px;
      font-weight: 600;
      margin-bottom: 10px;
    }

    .finance-dashboard .subtext {
      margin-top: 8px;
      color: var(--text-muted);
      font-size: 12px;
    }

    @media (max-width:520px) {
      .finance-dashboard {
        padding: 12px;
      }

      .finance-dashboard .title {
        font-size: 21px;
      }

      .finance-dashboard .number {
        font-size: 19px;
      }

      .finance-dashboard .stat,
      .finance-dashboard .week-card {
        padding: 10px;
      }
    }
  `;
  root.appendChild(style);

  function el(parent, tag, cls = "", content = "") {
    const node = document.createElement(tag);
    node.className = cls;
    node.textContent = String(content);
    parent.appendChild(node);
    return node;
  }

  function section(title) {
    const box = el(root, "div", "section");
    el(box, "div", "heading", title);
    return box;
  }

  function row(parent, left, right) {
    const box = el(parent, "div", "row");
    el(box, "span", "", left);
    el(box, "span", "muted", right);
  }

  function stat(parent, name, value, color = "") {
    const box = el(parent, "div", "stat");
    el(box, "div", "muted", name);
    el(box, "div", `number ${color}`, value);
  }

  function bar(parent, value, limit) {
    const percent = limit > 0 ? value / limit * 100 : 0;
    const track = el(parent, "div", "track");
    const fill = el(track, "div", "fill");
    fill.style.width = `${Math.min(100, Math.max(0, percent))}%`;
    fill.style.background = percent > 100 ? "#e16a6a" : percent >= 80 ? "#d99a35" : "#16a085";
  }

  function progress(parent, name, value, limit, rightText = null) {
    const box = el(parent, "div", "progress");
    row(box, name, rightText ?? `${money(value)} / ${money(limit)}`);
    bar(box, value, limit);
  }

  function categorySection(type, title, collapsible = false) {
    const names = Object.keys(limits).filter(name => (TYPES[name] || "flex") === type);
    if (!names.length) return;
    const parent = collapsible ? el(root, "div", "section") : section(title);
    let target = parent;
    if (collapsible) {
      const details = el(parent, "details", "fund-details");
      el(details, "summary", "heading", `${title} · ${names.length}`);
      target = el(details, "div");
    }
    for (const name of names) {
      const used = name === "Продукты" ? foodSpent : amount(costs.get(name)) ?? 0;
      const total = limits[name] + (previous[name] || 0);
      const label = `${ICONS[name] || "💳"} ${name}`;
      const info = carryNames.has(name) ? `${money(used)} / ${money(total)} · остаток ${money(total - used)}` : null;
      progress(target, label, used, total, info);
    }
  }

  const monthNames = ["Январь", "Февраль", "Март", "Апрель", "Май", "Июнь", "Июль", "Август", "Сентябрь", "Октябрь", "Ноябрь", "Декабрь"];
  const header = el(root, "div", "header");
  el(header, "div", "title", `💸 ${monthNames[month - 1]} ${year}`);
  el(header, "div", "muted", currency);

  const stats = el(root, "div", "stats");
  stat(stats, "Доход", money(income));
  stat(stats, "Остаток дохода после целей и трат", money(thisMonthRemaining), thisMonthRemaining >= 0 ? "green" : "red");

  const savingsBox = el(root, "div", "savings");
  row(savingsBox, "🔒 Подушка", money(reserve));
  el(savingsBox, "div", data.reserve_transferred === true ? "subtext green" : "subtext", data.reserve_transferred === true ? "✓ Переведено" : "○ К переводу");
  row(savingsBox, "✈️ Путешествия", money(travel));
  el(savingsBox, "div", data.travel_transferred === true ? "subtext green" : "subtext", data.travel_transferred === true ? "✓ Переведено" : "○ К переводу");

  const overall = el(root, "div", "progress");
  row(overall, "Расходы из зарплаты", `${income > reserve + travel ? Math.round(spent / (income - reserve - travel) * 100) : 0}%`);
  bar(overall, spent, income - reserve - travel);
  el(overall, "div", "subtext", `${money(spent)} из ${money(income - reserve - travel)}`);

  const plan = section("📊 План месяца");
  row(plan, "Категории", money(spendingPlan));
  row(plan, "Подушка", money(reserve));
  row(plan, "Путешествия", money(travel));
  row(plan, "Распределено", money(allocated));
  row(plan, free >= 0 ? "Свободный резерв" : "План превышает доход", money(Math.abs(free)));
  if (free < 0) el(plan, "div", "subtext red", "Уменьши лимиты категорий или суммы переводов.");

  const currentWeek = weeks.find(week => week.current);
  if (currentWeek) {
    const active = el(section("🛒 Продукты · текущая неделя"), "div", "active");
    el(active, "div", "week-title", dates(currentWeek.start));
    const weekStats = el(active, "div", "stats");
    stat(weekStats, "Потрачено", money(currentWeek.spent));
    stat(weekStats, "Доступно на неделю", money(currentWeek.available), currentWeek.available >= 0 ? "green" : "red");
    bar(active, currentWeek.spent, currentWeek.available);
    el(active, "div", "subtext", `Базовый ориентир: ${money(currentWeek.limit)} · ${currentWeek.carry >= 0 ? "Перенесено" : "Перерасход ранее"}: ${money(Math.abs(currentWeek.carry))}`);
    el(active, "div", `subtext ${currentWeek.leftover >= 0 ? "green" : "red"}`, `${currentWeek.leftover >= 0 ? "Осталось" : "Перерасход"}: ${money(Math.abs(currentWeek.leftover))}`);
    el(active, "div", "subtext", "Перенос действует только внутри месяца. В новом месяце лимит начинается заново.");
  }

  const weekGrid = el(section("📅 Недели месяца"), "div", "week-grid");
  for (const week of weeks) {
    const card = el(weekGrid, "div", `week-card ${week.current ? "current" : ""}`);
    el(card, "div", "week-title", dates(week.start));
    row(card, "Потрачено", `${money(week.spent)} / ${money(week.available)}`);
    bar(card, week.spent, week.available);
    if (week.visible) {
      if (Math.round(week.carry) !== 0) el(card, "div", `subtext ${week.carry > 0 ? "green" : "red"}`, `${week.carry > 0 ? "+" : "−"}${money(Math.abs(week.carry))} с прошлых недель`);
      el(card, "div", `subtext ${week.leftover >= 0 ? "green" : "red"}`, `${week.leftover >= 0 ? "Осталось" : "Перерасход"}: ${money(Math.abs(week.leftover))}`);
    }
  }

  categorySection("required", "🔒 Обязательные");
  categorySection("flex", "🎯 Гибкие");
  categorySection("fund", "🏦 Накопительные", true);

  const carried = section("🔄 Накопительные остатки на следующий месяц");
  for (const name of carryNames) {
    const value = balances[name];
    if (typeof value === "number" && (value !== 0 || limits[name])) row(carried, `${ICONS[name] || "💳"} ${name}`, money(value));
  }
  el(carried, "div", "subtext", "Отрицательное значение означает перерасход.");

  const result = section("🧮 Фактические расходы");
  row(result, "Продукты", money(foodSpent));
  row(result, "Остальные категории", money(otherSpent));
  row(result, "Всего внесено", money(spent));
  row(result, "Остаток текущей зарплаты", money(thisMonthRemaining));
  if (carryIn) row(result, "Остатки конвертов с предыдущих месяцев", money(carryIn));
  el(result, "div", "subtext", "Переносы — виртуальные конверты; проверь, что этим суммам соответствуют реальные деньги на счетах.");

  host.style.minHeight = `${root.getBoundingClientRect().height}px`;
  host.__financeLayout.observe(root);
}

await main();
```

## 📥 Расходы месяца

| Категория | Потрачено, ₽ |
|---|---:|

## 🛒 Продукты по неделям

| Неделя (Пн–Вс) | Потрачено, ₽ |
|---|---:|

## 🧠 Итоги месяца

**Удалось сэкономить:**

**Превысил лимит:**

**Что изменить в следующем месяце:**
