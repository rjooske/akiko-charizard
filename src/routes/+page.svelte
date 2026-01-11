<script lang="ts">
  import { browser } from "$app/environment";
  import { assert, sortByKeyCached } from "$lib/util";
  import courses2023Csv from "../courses/2023.csv?raw";
  import courses2024Csv from "../courses/2024.csv?raw";
  import courses2025Csv from "../courses/2025.csv?raw";
  import papaparse from "papaparse";
  import { SvelteSet } from "svelte/reactivity";

  type Course = {
    id: string;
    name: string;
    credit: string;
    term: string;
    when: string;
    expects: string;
    remark: string;
    description: string;
    organizer: string;
    year: number;
    key: string;
  };

  function courseCompare(a: Course, b: Course): number {
    if (a.key < b.key) {
      return -1;
    } else if (a.key > b.key) {
      return 1;
    } else {
      return 0;
    }
  }

  function courseContainsString(
    c: Course,
    s: string,
    useDescription: boolean,
  ): boolean {
    s = s.toLowerCase();
    return (
      c.id.toLowerCase().includes(s) ||
      c.name.toLowerCase().includes(s) ||
      c.credit.toLowerCase().includes(s) ||
      c.term.toLowerCase().includes(s) ||
      c.when.toLowerCase().includes(s) ||
      c.expects.toLowerCase().includes(s) ||
      c.remark.toLowerCase().includes(s) ||
      c.organizer.toLowerCase().includes(s) ||
      (useDescription && c.description.toLowerCase().includes(s))
    );
  }

  function parseCoursesCsv(csv: string, year: number): Course[] | undefined {
    const result = papaparse.parse(csv, { header: true });
    if (result.errors.length > 0) {
      return undefined;
    }

    const courses: Course[] = [];
    for (const row of result.data) {
      if (
        !(
          typeof row === "object" &&
          row !== null &&
          "科目番号" in row &&
          typeof row["科目番号"] === "string" &&
          "科目名" in row &&
          typeof row["科目名"] === "string" &&
          "単位数" in row &&
          typeof row["単位数"] === "string" &&
          "実施学期" in row &&
          typeof row["実施学期"] === "string" &&
          "曜時限" in row &&
          typeof row["曜時限"] === "string" &&
          "標準履修年次" in row &&
          typeof row["標準履修年次"] === "string" &&
          "備考" in row &&
          typeof row["備考"] === "string" &&
          "授業概要" in row &&
          typeof row["授業概要"] === "string" &&
          "担当教員" in row &&
          typeof row["担当教員"] === "string"
        )
      ) {
        continue;
      }
      const id = row["科目番号"].trim();
      if (id === "") {
        continue;
      }
      courses.push({
        id,
        name: row["科目名"].trim(),
        credit: row["単位数"].trim(),
        term: row["実施学期"].trim(),
        when: row["曜時限"].trim(),
        expects: row["標準履修年次"].trim(),
        remark: row["備考"].trim(),
        description: row["授業概要"].trim(),
        organizer: row["担当教員"].trim(),
        year,
        key: `${id},${year}`,
      });
    }

    return courses;
  }

  function loadCourses(): Course[] {
    const courses: Course[] = [];
    for (const [csv, year] of [
      [courses2023Csv, 2023],
      [courses2024Csv, 2024],
      [courses2025Csv, 2025],
    ] as const) {
      const cs = parseCoursesCsv(csv, year);
      assert(cs !== undefined);
      for (const c of cs) {
        courses.push(c);
      }
    }
    courses.sort(courseCompare);
    return courses;
  }

  const courses = loadCourses();

  let idFilter = $state("");
  let idFilterMode = $state<"prefix" | "contain">("prefix");
  let nameFilter = $state("");
  let nameFilterMode = $state<"contain" | "exact">("contain");
  let yearFilter = $state(new SvelteSet([2023, 2024, 2025]));
  let priorityString = $state("");
  let showDescription = $state(false);
  let maxVisibleCourseCount = $state(1000);

  let [visibleCourses, visibleCourseIdRowSpans, maxExceeded, prioritizedCount] =
    $derived.by(() => {
      const idPrefix = idFilterMode === "prefix";
      const nameExact = nameFilterMode === "exact";
      const visibleCourses: Course[] = [];
      const uppercaseIdFilter = idFilter.toUpperCase();
      const lowercaseNameFilter = nameFilter.toLowerCase();
      let maxExceeded = false;
      for (const c of courses) {
        if (
          !(
            (idPrefix
              ? c.id.startsWith(uppercaseIdFilter)
              : c.id.includes(uppercaseIdFilter)) &&
            (nameExact
              ? c.name === nameFilter
              : c.name.toLowerCase().includes(lowercaseNameFilter)) &&
            yearFilter.has(c.year)
          )
        ) {
          continue;
        }
        visibleCourses.push(c);
        if (visibleCourses.length === maxVisibleCourseCount + 1) {
          visibleCourses.pop();
          maxExceeded = true;
          break;
        }
      }

      let prioritizedCount: number | undefined;
      if (priorityString !== "") {
        sortByKeyCached(
          visibleCourses,
          (c) => -courseContainsString(c, priorityString, showDescription),
        );
        prioritizedCount = 0;
        for (const c of visibleCourses) {
          if (courseContainsString(c, priorityString, showDescription)) {
            prioritizedCount++;
          } else {
            break;
          }
        }
      }

      const duplicateCourseIdCounts: number[] = [];
      let lastCourseId: string | undefined;
      for (const c of visibleCourses) {
        if (lastCourseId !== c.id) {
          lastCourseId = c.id;
          duplicateCourseIdCounts.push(0);
        }
        duplicateCourseIdCounts[duplicateCourseIdCounts.length - 1]++;
      }

      const visibleCourseIdRowSpans: number[] = [];
      for (const count of duplicateCourseIdCounts) {
        visibleCourseIdRowSpans.push(count);
        for (let i = 0; i < count - 1; i++) {
          visibleCourseIdRowSpans.push(0);
        }
      }

      return [
        visibleCourses,
        visibleCourseIdRowSpans,
        maxExceeded,
        prioritizedCount,
      ];
    });

  let idElement = $state<HTMLElement | undefined>();
  let nameElement = $state<HTMLElement | undefined>();
  let priorityStringElement = $state<HTMLElement | undefined>();

  if (browser) {
    window.addEventListener("keydown", (e) => {
      if (e.ctrlKey) {
        if (e.code === "KeyI") {
          idElement?.focus();
        }
        if (e.code === "KeyN") {
          nameElement?.focus();
        }
        if (e.code === "KeyP") {
          priorityStringElement?.focus();
        }
        if (e.code === "KeyY") {
          let s = "";
          for (let i = 0; i < visibleCourseIdRowSpans.length; i++) {
            const rowSpan = visibleCourseIdRowSpans[i];
            if (rowSpan === 0) {
              continue;
            }
            const course = visibleCourses[i + rowSpan - 1];
            s += course.id + " " + course.name + "\n";
          }
          window.navigator.clipboard.writeText(s);
        }
      }
    });
  }
</script>

<h1>
  <img src="akiko-charizard.png" alt="あきこリザードン" />
  <span>あきこリザードン</span>
</h1>

<label bind:this={idElement}>
  <kbd>Ctrl + I</kbd>
  科目番号：
  <input type="text" bind:value={idFilter} />
</label>
<select bind:value={idFilterMode}>
  <option value={"prefix"}>先頭一致</option>
  <option value={"contain"}>部分一致</option>
</select>
<br />
<label bind:this={nameElement}>
  <kbd>Ctrl + N</kbd>
  科目名：
  <input type="text" bind:value={nameFilter} />
</label>
<select bind:value={nameFilterMode}>
  <option value={"contain"}>部分一致</option>
  <option value={"exact"}>完全一致</option>
</select>
<br />
{#each [2023, 2024, 2025] as year}
  <label class="year">
    <input
      type="checkbox"
      checked={yearFilter.has(year)}
      onchange={(e) => {
        if (e.currentTarget.checked) {
          yearFilter.add(year);
        } else {
          yearFilter.delete(year);
        }
      }}
    />
    {year}年度
  </label>
{/each}
<br />
<label bind:this={priorityStringElement}>
  <kbd>Ctrl + P</kbd>
  優先する文字列：
  <input type="text" bind:value={priorityString} />
  {#if priorityString !== "" && maxExceeded}
    （⚠️ 表示されている{maxVisibleCourseCount}授業のみ並べ替えます）
  {/if}
</label>
<br />
<label>
  授業概要を表示：
  <input type="checkbox" bind:checked={showDescription} />
</label>
<br />
<br />

<span>該当授業数：{visibleCourses.length}{maxExceeded ? "以上" : ""}</span>
<br />
<label>
  最大表示数：
  <input type="number" bind:value={maxVisibleCourseCount} />
</label>
<br />
<br />
<br />

<table class={{ "hide-desc": !showDescription }}>
  <thead>
    <tr>
      <th>科目番号</th>
      <th>
        年度
        <br />
        科目名
      </th>
      <th>
        単位数
        <br />
        実施学期
        <br />
        曜時限
      </th>
      <th>
        標準履修年次
        <br />
        備考
        <br />
        担当教員
      </th>
      <th class="desc">授業概要</th>
      {#if prioritizedCount !== undefined}
        <th>優先</th>
      {/if}
    </tr>
  </thead>
  <tbody>
    {#each visibleCourses as c, i (c.key)}
      {@const rowSpan = visibleCourseIdRowSpans[i]}
      <tr data-year={c.year}>
        {#if rowSpan > 0}
          <td rowspan={rowSpan} class="id">
            {c.id}
            <br />
            <button onclick={() => window.navigator.clipboard.writeText(c.id)}>
              コピー
            </button>
          </td>
        {/if}
        <td>
          {c.year}
          <br />
          <a
            href="https://kdb.tsukuba.ac.jp/syllabi/{c.year}/{c.id}/jpn"
            target="_blank"
            rel="noreferrer"
          >
            {c.name}
          </a>
          <br />
          <button
            onclick={() => window.navigator.clipboard.writeText(c.name)}
            style="text-wrap: nowrap;"
          >
            科目名コピー
          </button>
        </td>
        <td>
          {c.credit}
          <br />
          {c.term}
          <br />
          {c.when}
        </td>
        <td>
          {c.expects}
          <br />
          {c.remark}
          <br />
          {c.organizer}
        </td>
        <td class="desc">{c.description}</td>
        {#if prioritizedCount !== undefined}
          <td>{i < prioritizedCount ? "✅" : ""}</td>
        {/if}
      </tr>
    {/each}
  </tbody>
</table>

<style lang="scss">
  :global(body) {
    font-family: sans-serif;
  }

  h1 {
    display: flex;
    align-items: center;
    gap: 20px;

    & > img {
      width: 50px;
    }
  }

  label > kbd {
    background-color: oklch(92% 0 0);
    border-radius: 5px;
    padding: 0 5px;
  }

  label.year {
    & + & {
      margin-left: 20px;
    }
  }

  table {
    width: 100%;
  }

  table,
  th,
  td {
    border: 1px solid black;
    border-collapse: collapse;
  }

  th {
    text-wrap: nowrap;
  }

  td.desc {
    font-size: 12px;
  }

  table.hide-desc {
    & th,
    & td {
      &.desc {
        display: none;
      }
    }
  }

  tr {
    $l: 98%;
    $c: 8%;
    &[data-year="2023"] {
      background-color: oklch($l $c 0);
    }
    &[data-year="2024"] {
      background-color: oklch($l $c 120);
    }
    &[data-year="2025"] {
      background-color: oklch($l $c 240);
    }
  }
</style>
