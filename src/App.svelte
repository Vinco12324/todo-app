<script>
  const categories = ["Personal", "Work", "Study", "Shopping"];
  const priorities = ["Low", "Medium", "High"];

  function load(key, fallback) {
    try {
      const saved = localStorage.getItem(key);
      return saved ? JSON.parse(saved) : fallback;
    } catch {
      return fallback;
    }
  }

  let tasks = $state(load("tasks", []));
  let dark = $state(load("dark", false));

  let text = $state("");
  let category = $state("Personal");
  let priority = $state("Medium");
  let dueDate = $state("");

  let filter = $state("all");
  let search = $state("");

  let editingId = $state(null);
  let editText = $state("");

  // Save automatically
  $effect(() => {
    localStorage.setItem("tasks", JSON.stringify(tasks));
  });
  $effect(() => {
    localStorage.setItem("dark", JSON.stringify(dark));
  });

  // Calculated values
  let total = $derived(tasks.length);
  let doneCount = $derived(tasks.filter((t) => t.done).length);
  let percent = $derived(total === 0 ? 0 : Math.round((doneCount / total) * 100));

  let visible = $derived(
    tasks.filter((t) => {
      if (filter === "active" && t.done) return false;
      if (filter === "completed" && !t.done) return false;
      const q = search.trim().toLowerCase();
      if (q && !t.text.toLowerCase().includes(q)) return false;
      return true;
    })
  );

  function addTask() {
    if (text.trim() === "") return;
    tasks.unshift({
      id: Date.now(),
      text: text.trim(),
      done: false,
      category,
      priority,
      dueDate,
    });
    text = "";
    dueDate = "";
  }

  function toggleTask(id) {
    const t = tasks.find((t) => t.id === id);
    if (t) t.done = !t.done;
  }

  function removeTask(id) {
    tasks = tasks.filter((t) => t.id !== id);
  }

  function startEdit(task) {
    editingId = task.id;
    editText = task.text;
  }

  function saveEdit(id) {
    const t = tasks.find((t) => t.id === id);
    if (t && editText.trim() !== "") t.text = editText.trim();
    editingId = null;
  }

  function clearCompleted() {
    tasks = tasks.filter((t) => !t.done);
  }

  function isOverdue(task) {
    const today = new Date().toLocaleDateString("en-CA");
    return task.dueDate && !task.done && task.dueDate < today;
  }

  function formatDate(d) {
    return new Date(d + "T00:00:00").toLocaleDateString(undefined, {
      month: "short",
      day: "numeric",
      year: "numeric",
    });
  }
</script>

<div class="app" class:dark>
  <div class="container">
    <header>
      <div>
        <h1>✅ TaskFlow</h1>
        <p class="subtitle">Plan your day, one task at a time.</p>
      </div>
      <button class="icon-btn" onclick={() => (dark = !dark)} title="Toggle dark mode">
        {dark ? "☀️" : "🌙"}
      </button>
    </header>

    <section class="card">
      <div class="progress-text">
        <span>{doneCount} of {total} completed</span>
        <span>{percent}%</span>
      </div>
      <div class="bar"><div class="fill" style="width: {percent}%"></div></div>
    </section>

    <form
      class="card add"
      onsubmit={(e) => {
        e.preventDefault();
        addTask();
      }}
    >
      <input class="text-input" bind:value={text} placeholder="What do you need to do?" />
      <div class="row">
        <select bind:value={category}>
          {#each categories as c}
            <option value={c}>{c}</option>
          {/each}
        </select>
        <select bind:value={priority}>
          {#each priorities as p}
            <option value={p}>{p} priority</option>
          {/each}
        </select>
        <input type="date" bind:value={dueDate} />
        <button class="primary" type="submit">Add Task</button>
      </div>
    </form>

    <section class="toolbar">
      <div class="tabs">
        {#each ["all", "active", "completed"] as f}
          <button class="tab" class:active={filter === f} onclick={() => (filter = f)}>
            {f[0].toUpperCase() + f.slice(1)}
          </button>
        {/each}
      </div>
      <input class="search" type="search" bind:value={search} placeholder="Search tasks..." />
    </section>

    <ul>
      {#each visible as task (task.id)}
        <li class="card task" class:done={task.done}>
          <input type="checkbox" checked={task.done} onchange={() => toggleTask(task.id)} />

          <div class="task-body">
            {#if editingId === task.id}
              <input
                class="edit-input"
                bind:value={editText}
                onkeydown={(e) => {
                  if (e.key === "Enter") saveEdit(task.id);
                  if (e.key === "Escape") editingId = null;
                }}
              />
            {:else}
              <span class="task-text">{task.text}</span>
            {/if}

            <div class="meta">
              <span class="badge">{task.category}</span>
              <span
                class="badge"
                class:low={task.priority === "Low"}
                class:medium={task.priority === "Medium"}
                class:high={task.priority === "High"}
              >
                {task.priority}
              </span>
              {#if task.dueDate}
                <span class="due" class:overdue={isOverdue(task)}>
                  📅 {formatDate(task.dueDate)}{isOverdue(task) ? " (overdue)" : ""}
                </span>
              {/if}
            </div>
          </div>

          <div class="actions">
            {#if editingId === task.id}
              <button onclick={() => saveEdit(task.id)}>Save</button>
              <button onclick={() => (editingId = null)}>Cancel</button>
            {:else}
              <button onclick={() => startEdit(task)} title="Edit">✏️</button>
              <button onclick={() => removeTask(task.id)} title="Delete">🗑️</button>
            {/if}
          </div>
        </li>
      {/each}
    </ul>

    {#if total === 0}
      <p class="empty">🎉 No tasks yet. Add your first one above!</p>
    {:else if visible.length === 0}
      <p class="empty">No tasks match this view.</p>
    {/if}

    {#if doneCount > 0}
      <button class="clear" onclick={clearCompleted}>Clear completed ({doneCount})</button>
    {/if}
  </div>
</div>

<style>
  .app {
    --bg: #f3f4f8;
    --card: #ffffff;
    --text: #1f2430;
    --muted: #7a8194;
    --border: #e3e6ee;
    --primary: #4f46e5;
    min-height: 100vh;
    background: var(--bg);
    color: var(--text);
    padding: 2rem 1rem;
  }
  .app.dark {
    --bg: #12141c;
    --card: #1d212d;
    --text: #e8eaf2;
    --muted: #9aa2b8;
    --border: #2d3345;
    --primary: #7c75ff;
  }

  .container {
    max-width: 640px;
    margin: 0 auto;
  }

  header {
    display: flex;
    justify-content: space-between;
    align-items: center;
    margin-bottom: 1.2rem;
  }
  h1 {
    margin: 0;
    font-size: 1.9rem;
  }
  .subtitle {
    margin: 0.2rem 0 0;
    color: var(--muted);
  }
  .icon-btn {
    font-size: 1.3rem;
    width: 44px;
    height: 44px;
    border-radius: 50%;
  }

  .card {
    background: var(--card);
    border: 1px solid var(--border);
    border-radius: 12px;
    padding: 1rem;
    margin-bottom: 0.8rem;
  }

  .progress-text {
    display: flex;
    justify-content: space-between;
    font-size: 0.9rem;
    color: var(--muted);
    margin-bottom: 0.5rem;
  }
  .bar {
    height: 10px;
    background: var(--border);
    border-radius: 999px;
    overflow: hidden;
  }
  .fill {
    height: 100%;
    background: var(--primary);
    transition: width 0.3s;
  }

  .add {
    display: flex;
    flex-direction: column;
    gap: 0.7rem;
  }
  .row {
    display: flex;
    flex-wrap: wrap;
    gap: 0.5rem;
  }

  input,
  select,
  button {
    font-family: inherit;
    font-size: 0.95rem;
    color: var(--text);
    background: var(--card);
    border: 1px solid var(--border);
    border-radius: 8px;
    padding: 0.55rem 0.7rem;
  }
  button {
    cursor: pointer;
  }
  button:hover {
    border-color: var(--primary);
  }
  .text-input {
    width: 100%;
    font-size: 1.05rem;
  }
  .primary {
    background: var(--primary);
    border-color: var(--primary);
    color: white;
    font-weight: 600;
    margin-left: auto;
  }

  .toolbar {
    display: flex;
    flex-wrap: wrap;
    gap: 0.5rem;
    justify-content: space-between;
    margin: 1rem 0;
  }
  .tabs {
    display: flex;
    gap: 0.4rem;
  }
  .tab.active {
    background: var(--primary);
    border-color: var(--primary);
    color: white;
  }
  .search {
    flex: 1;
    min-width: 140px;
  }

  ul {
    list-style: none;
    padding: 0;
    margin: 0;
  }
  .task {
    display: flex;
    align-items: flex-start;
    gap: 0.8rem;
  }
  .task input[type="checkbox"] {
    width: 20px;
    height: 20px;
    margin-top: 0.2rem;
    cursor: pointer;
  }
  .task-body {
    flex: 1;
  }
  .task-text {
    font-size: 1.05rem;
    word-break: break-word;
  }
  .task.done .task-text {
    text-decoration: line-through;
    color: var(--muted);
  }
  .edit-input {
    width: 100%;
  }

  .meta {
    display: flex;
    flex-wrap: wrap;
    gap: 0.4rem;
    margin-top: 0.4rem;
    font-size: 0.8rem;
  }
  .badge {
    padding: 0.15rem 0.6rem;
    border-radius: 999px;
    background: var(--border);
    color: var(--muted);
  }
  .badge.low {
    background: #d9f5e3;
    color: #1d7a45;
  }
  .badge.medium {
    background: #fff1cc;
    color: #8a6100;
  }
  .badge.high {
    background: #ffdcdc;
    color: #b42323;
  }
  .due {
    color: var(--muted);
    padding: 0.15rem 0;
  }
  .due.overdue {
    color: #d93636;
    font-weight: 600;
  }

  .actions {
    display: flex;
    gap: 0.3rem;
  }
  .actions button {
    padding: 0.3rem 0.6rem;
  }

  .empty {
    text-align: center;
    color: var(--muted);
    margin: 2rem 0;
  }
  .clear {
    display: block;
    margin: 1rem auto 0;
  }
</style>