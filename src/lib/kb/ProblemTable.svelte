<script lang="ts">
  import reports from '#lib/data/reports-4471.json';
  import notes from '#lib/data/desk-notes.json';
  import { causeOf, uniqueById, type Report } from '#lib/kb/causes';
  import { tenantOf } from '#lib/kb/locales';

  type Problem = 'parser' | 'hint' | 'one-day move';
  let { ticket, filed }: { ticket: number; filed: Record<Problem, string> } = $props();

  const LABEL: Record<Problem, string> = {
    parser: 'shared parser reads day-first dates month first',
    hint: 'booking-v3 form hint says MM/DD, form reads DD/MM',
    'one-day move': 'date moves by one day (time zone), not a swap',
  };

  // Counted the desk's way: every report under its cause by tenant; one-day moves stay there too
  // and are also counted on their own row (desk-notes.json), as CauseTable footnotes them.
  function inProblem(r: Report, p: Problem): boolean {
    return p === 'one-day move' ? notes.oneDayMove.includes(r.id) : causeOf(r, tenantOf(r)) === p;
  }

  const merged = $derived(uniqueById(reports as Report[]).filter((r) => r.mergedInto === ticket));
  const rows = $derived(
    (Object.keys(LABEL) as Problem[]).map((p) => {
      const rs = merged.filter((r) => inProblem(r, p));
      const days = rs.map((r) => r.receivedAt.slice(0, 10)).sort();
      return { p, count: rs.length, tenants: new Set(rs.map((r) => r.tenant)).size, first: days[0], last: days[days.length - 1] };
    }),
  );
  const placed = $derived(merged.filter((r) => causeOf(r, tenantOf(r)) !== null).length);
  const alsoParser = $derived(merged.filter((r) => inProblem(r, 'one-day move') && inProblem(r, 'parser')).length);
</script>

<p class="sum">
  <strong>{merged.length}</strong> reports in #{ticket} · <strong>{rows.length}</strong> problems ·
  {placed === merged.length ? 'every report placed' : `${merged.length - placed} not placed`}
</p>

<table>
  <thead><tr><th>problem</th><th>ticket</th><th>reports</th><th>tenants</th><th>seen</th></tr></thead>
  <tbody>
    {#each rows as r (r.p)}
      <tr>
        <td><code>{r.p}</code> {LABEL[r.p]}</td>
        <td><a href={filed[r.p]}>{filed[r.p].replace(/^.*\/issues\//, '#')}</a></td>
        <td>{r.count}</td>
        <td>{r.tenants}</td>
        <td>{r.first} → {r.last}</td>
      </tr>
    {/each}
  </tbody>
</table>

{#if alsoParser}
  <p class="note">{alsoParser} one-day moves are also counted under <code>parser</code>, where the desk files them by tenant.</p>
{/if}

<style>
  .sum { font-size: 1.1rem; }
  .note { color: var(--muted); font-size: 0.85em; }
</style>
