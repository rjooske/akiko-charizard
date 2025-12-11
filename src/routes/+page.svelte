<script lang="ts">
  import { assert, sortByKeyCached } from "$lib/util";
  import courses2023Csv from "../courses/2023.csv?raw";
  import courses2024Csv from "../courses/2024.csv?raw";
  import courses2025Csv from "../courses/2025.csv?raw";
  import papaparse from "papaparse";

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

  function courseContainsString(c: Course, s: string): boolean {
    return (
      c.id.includes(s) ||
      c.name.includes(s) ||
      c.credit.includes(s) ||
      c.term.includes(s) ||
      c.when.includes(s) ||
      c.expects.includes(s) ||
      c.remark.includes(s) ||
      c.description.includes(s) ||
      c.organizer.includes(s)
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
  let yearFilter = $state<number | undefined>();
  let priorityString = $state("");
  let maxVisibleCourseCount = $state(1000);

  let [visibleCourses, maxExceeded, prioritizedCount] = $derived.by(() => {
    const prefix = idFilterMode === "prefix";
    const visibleCourses: Course[] = [];
    let maxExceeded = false;
    for (const c of courses) {
      if (
        !(
          (prefix ? c.id.startsWith(idFilter) : c.id.includes(idFilter)) &&
          c.name.includes(nameFilter) &&
          (yearFilter === undefined || c.year === yearFilter)
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
        (c) => -courseContainsString(c, priorityString),
      );
      prioritizedCount = 0;
      for (const c of visibleCourses) {
        if (courseContainsString(c, priorityString)) {
          prioritizedCount++;
        } else {
          break;
        }
      }
    }

    return [visibleCourses, maxExceeded, prioritizedCount];
  });
</script>

<h1>
  <img src="akiko-charizard.png" alt="あきこリザードン" />
  <span>あきこリザードン</span>
</h1>

<label>
  科目番号：
  <input type="text" bind:value={idFilter} />
</label>
<select bind:value={idFilterMode}>
  <option value={"prefix"}>先頭一致</option>
  <option value={"contain"}>部分一致</option>
</select>
<br />
<label>
  科目名：
  <input type="text" bind:value={nameFilter} />
</label>
<br />
<label>
  年度：
  <input
    type="number"
    oninput={(event) => {
      const year = parseInt(event.currentTarget.value);
      yearFilter = isNaN(year) ? undefined : year;
    }}
  />
</label>
<br />
<label>
  優先する文字列：
  <input type="text" bind:value={priorityString} />
  {#if priorityString !== "" && maxExceeded}
    （⚠️ 表示されている{maxVisibleCourseCount}授業のみ並べ替えます）
  {/if}
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

<table>
  <thead>
    <tr>
      <th>
        年度
        <br />
        科目番号
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
      <th>授業概要</th>
      {#if prioritizedCount !== undefined}
        <th>優先</th>
      {/if}
    </tr>
  </thead>
  <tbody>
    {#each visibleCourses as c, i (c.key)}
      <tr data-year={c.year}>
        <td>
          {c.year}
          <br />
          {c.id}
          <br />
          {c.name}
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
        <td>{c.description}</td>
        {#if prioritizedCount !== undefined}
          <td>{i < prioritizedCount ? "✅" : ""}</td>
        {/if}
      </tr>
    {/each}
  </tbody>
</table>

<style lang="scss">
  h1 {
    display: flex;
    align-items: center;
    gap: 20px;

    & > img {
      width: 50px;
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

  th,
  td {
    &:nth-child(1) {
      max-width: 40ch;
    }
    &:nth-child(2) {
      max-width: 10ch;
    }
    &:nth-child(3) {
      max-width: 80ch;
    }
  }

  th {
    text-wrap: nowrap;
  }

  td:nth-child(4) {
    font-size: 12px;
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
