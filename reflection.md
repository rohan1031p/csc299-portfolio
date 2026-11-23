# CSC 299 Reflection

All file, line and commit references are to the expense-tracker repo: https://github.com/rohan1031p/expense-tracker

## 1. Fresh session

In the Expense Tracker project, the fresh Claude Code session for Round 2 had to stop and ask me about the decision records before it could start. Round 1 had named them `001` and `002`, but my Round 2 prompt asked for four-digit numbers (`0001`, `0002`), so the session didn't know whether to rename the old files or keep the old style. It also found that the Round 1 records were not written as Y-statements, which the new rounds required. This happened because Round 1 had no handoff file, and `CLAUDE.md` said nothing about how decision records should be named or written, so the Round 1 choices were never written down as rules. I could have prevented this by adding the naming rule and the Y-statement format to `CLAUDE.md` in Round 1. I added both at the end of Round 2, so later sessions followed them without asking.

Reference: commit `033d45c` (renamed the records after the session asked), commit `e84d68d` (added the rules to `CLAUDE.md`), `CLAUDE.md` lines 58–59.

## 2. Changed requirement

In Round 4, I changed requirement R5 to show spending in each category compared to its budget. When I made this change, the category box went from adding up every expense ever entered to showing only this month's spending, and each row now shows "(no budget)" or "of $X". This broke three existing browser tests (R5, R12 and R14) that expected the old all-time totals, like "Food $50.00". The tests caught the problem because they compare the category rows as exact text, so they failed until I updated them to the new this-month format. The decision record explains why I changed the requirement: budgets are monthly, so comparing them with all-time spending could make a category look over budget even when its spending for the current month is within its budget.

Reference: test "R5 (browser): spending this month is shown for each category, without a budget", `tests/browser.test.js` line 171; `docs/decisions/0006-category-spending-this-month.md` lines 7–12 and 19; commit `a6e6d32`.

## 3. A useful test

One test that caught a real problem was:

```js
test("Layout (browser): on a phone-sized screen each expense is a card with labels and visible Edit and Delete buttons", async () => {
  await page.setViewportSize({ width: 375, height: 800 });
  await addExpense("A long expense name for testing", "1234.56", "Entertainment", await today());

  // The column headings are hidden; each value shows its own label instead.
  assert.equal(await page.isVisible("#expense-table thead"), false);
  const label = await page.$eval("#expense-list td:nth-child(2)", (td) =>
    getComputedStyle(td, "::before").content
  );
  assert.equal(label, '"Name"');

  // The Edit and Delete buttons are fully inside the screen, without scrolling sideways.
  for (const selector of ["#expense-list button.edit", "#expense-list button.delete"]) {
    const box = await page.locator(selector).boundingBox();
    assert.ok(box.x >= 0 && box.x + box.width <= 375, selector + " is at x=" + box.x);
  }

  // The card still works: Delete removes the expense.
  await page.click("#expense-list button.delete");
  assert.equal(await rowCount(), 0);
});
```

This test was useful because it caught a real bug on phone-sized screens. In the old version of the app, the Delete button sat just off the right edge of a 375px screen (x = 376.6), so a phone user couldn't reach it without scrolling the table sideways. My earlier layout test, "the page does not scroll sideways," still passed on that version because the table scrolled inside its own box. This test checks where the Edit and Delete buttons actually appear, so it fails on the old code and passes after the fix. It also checks that the labels show and that Delete still works on the card layout.

Reference: `tests/browser.test.js` line 1259 (added in commit `5aa6981`).

## 4. A commit I would change

One commit I would write differently is `a6e6d32` "Tests for category budgets (R17, R18) and the changed R5". I would split it into two commits: one that updates the existing R5, R12 and R14 tests for the changed requirement, and one that adds the new R17 and R18 tests. This would make the project easier to understand because someone reading the history could see exactly which old behavior changed, separately from the 300+ lines of new tests.

## 5. Understanding a function

The function I chose is `getCategorySummary` in `expenses.js` (line 209).

Without using an assistant, I think this function looks at expenses from a selected month and adds up how much was spent in each category. It compares each category's spending with its budget and returns a list showing the spending, budget, whether it went over, and how much it went over.

After asking an assistant, I learned that my main idea was right, but I missed some details. The function skips categories that have no spending that month and no budget, and it uses `null` for a category with no budget, so it is not marked as over budget. Spending exactly equal to the budget does not count as over because it checks `spent > budget`. Also, `overBy` is 0 when a category is not over, and the list always follows the fixed category order. In Round 6, the function also got a `left` field, which shows how much of the budget is left, or `null` when there is no budget or the category is over.