# File 1 — read (read.md)

---

# Expense Tracker — Project Script

## Title

Personal Expense Tracker (CLI + CSV storage + reports)

## Introduction

Managing personal finances is easier when you can quickly record expenses, review spending patterns, and generate simple reports. This project implements a lightweight, easy-to-run **Personal Expense Tracker** written in Python. It stores data in a CSV file, provides a CLI interface for adding and viewing expenses, and produces summary reports (total spent, category breakdowns, monthly summaries) and visualizations.

## Objectives

* Provide a simple CLI to add, list, and analyze expenses.
* Persist expenses in a human-readable CSV so users can edit or back them up easily.
* Offer quick summary reports (by month, by category) and optional plots.
* Keep the code beginner-friendly but modular enough for expansion (GUI, DB, authentication, etc.).

## Features

1. Add expense record (date, amount, category, description).
2. List recent expenses and search by date or category.
3. Monthly summary (total, average daily spend).
4. Category breakdown with percentages.
5. Export filtered data to CSV.
6. Simple plots (spending by category, spending over time) using matplotlib.
7. Data validation and helpful error messages.

## Use Cases

* Students tracking pocket money and subscriptions.
* Freelancers monitoring work-related expenses.
* Anyone who wants a simple offline tool to log and inspect spending.

## Impact

This tool helps users become more aware of their spending patterns, identify categories where they overspend, and plan budgets. Because data is stored locally as CSV, privacy is preserved and users retain full control of their records.

## Implementation Details (high-level)

* Language: Python 3.8+
* Storage: `expenses.csv` (CSV with header: id,date,amount,category,description)
* Libraries: `csv`, `datetime`, `argparse`, `matplotlib` (optional), `collections`, `statistics`.
* Structure: modular functions for add/list/report/plot; a `main()` that parses CLI args.

## Extensibility Ideas

* Add user authentication and encrypted storage.
* Provide a web dashboard (Flask/FastAPI) or a GUI (Tkinter, PySimpleGUI).
* Sync with cloud storage or allow backups to Google Drive/Dropbox.
* Auto-categorization using ML/NLP of descriptions.

## Conclusion

This project is intentionally small and practical — a great starting point to learn file I/O, data aggregation, CLI design, and basic plotting in Python. It can be used as-is by people who need a lightweight expense tracker or extended into a fully featured personal finance app.

---

# File 2 — project (project.py)

```python
"""
Personal Expense Tracker
Author: generated for user

Usage examples:
  python project.py add --amount 250 --category Food --description "lunch" --date 2025-11-01
  python project.py list --limit 10
  python project.py report --month 2025-11
  python project.py plot --type category

Files created/used:
  expenses.csv

Requires:
  matplotlib (for plots; optional)
"""

import csv
import argparse
from datetime import datetime, date
from pathlib import Path
from collections import defaultdict, Counter
import statistics
import sys

CSV_FILE = Path("expenses.csv")
CSV_HEADER = ["id", "date", "amount", "category", "description"]


def ensure_csv():
    if not CSV_FILE.exists():
        with CSV_FILE.open("w", newline='', encoding='utf-8') as f:
            writer = csv.writer(f)
            writer.writerow(CSV_HEADER)


def generate_id():
    ensure_csv()
    with CSV_FILE.open('r', newline='', encoding='utf-8') as f:
        reader = csv.DictReader(f)
        ids = [int(row['id']) for row in reader if row.get('id') and row['id'].isdigit()]
    return max(ids) + 1 if ids else 1


def add_expense(amount, category, description, date_str=None):
    ensure_csv()
    if date_str:
        try:
            d = datetime.strptime(date_str, "%Y-%m-%d").date()
        except ValueError:
            print("Date must be in YYYY-MM-DD format.")
            return
    else:
        d = date.today()

    try:
        amt = float(amount)
    except ValueError:
        print("Amount must be a number.")
        return

    rec_id = generate_id()
    with CSV_FILE.open('a', newline='', encoding='utf-8') as f:
        writer = csv.writer(f)
        writer.writerow([rec_id, d.isoformat(), f"{amt:.2f}", category.strip(), description.strip()])
    print(f"Added expense id={rec_id} date={d} amount={amt:.2f} category={category}")


def read_all():
    ensure_csv()
    with CSV_FILE.open('r', newline='', encoding='utf-8') as f:
        reader = csv.DictReader(f)
        return list(reader)


def list_expenses(limit=None, category=None, start_date=None, end_date=None):
    rows = read_all()
    if category:
        rows = [r for r in rows if r['category'].lower() == category.lower()]
    if start_date:
        rows = [r for r in rows if r['date'] >= start_date]
    if end_date:
        rows = [r for r in rows if r['date'] <= end_date]

    rows_sorted = sorted(rows, key=lambda r: r['date'], reverse=True)
    if limit:
        rows_sorted = rows_sorted[:limit]

    if not rows_sorted:
        print("No expenses found.")
        return

    print(f"Showing {len(rows_sorted)} expenses:\n")
    for r in rows_sorted:
        print(f"{r['id']:>4} | {r['date']} | {r['amount']:>8} | {r['category']:<12} | {r['description']}")


def monthly_summary(year_month):
    # year_month: YYYY-MM
    rows = read_all()
    filtered = [r for r in rows if r['date'].startswith(year_month)]
    if not filtered:
        print(f"No expenses for {year_month}.")
        return

    amounts = [float(r['amount']) for r in filtered]
    total = sum(amounts)
    avg = statistics.mean(amounts)
    per_category = defaultdict(float)
    for r in filtered:
        per_category[r['category']] += float(r['amount'])

    print(f"Summary for {year_month}:")
    print(f"  Total spent: {total:.2f}")
    print(f"  Transactions: {len(amounts)}")
    print(f"  Average transaction: {avg:.2f}\n")
    print("  By category:")
    for cat, amt in sorted(per_category.items(), key=lambda x: x[1], reverse=True):
        pct = (amt / total) * 100 if total else 0
        print(f"    {cat:<12} {amt:>8.2f} ({pct:>5.1f}%)")


def report_range(start_date=None, end_date=None):
    rows = read_all()
    if start_date:
        rows = [r for r in rows if r['date'] >= start_date]
    if end_date:
        rows = [r for r in rows if r['date'] <= end_date]
    if not rows:
        print("No expenses in selected range.")
        return
    total = sum(float(r['amount']) for r in rows)
    print(f"Report from {start_date or 'start'} to {end_date or 'end'}")
    print(f"  Total: {total:.2f}")
    cats = Counter(r['category'] for r in rows)
    print("  Category counts:")
    for cat, cnt in cats.most_common():
        print(f"    {cat:<12} {cnt}")


def export_csv(out_file, start_date=None, end_date=None, category=None):
    rows = read_all()
    if category:
        rows = [r for r in rows if r['category'].lower() == category.lower()]
    if start_date:
        rows = [r for r in rows if r['date'] >= start_date]
    if end_date:
        rows = [r for r in rows if r['date'] <= end_date]

    with open(out_file, 'w', newline='', encoding='utf-8') as f:
        writer = csv.writer(f)
        writer.writerow(CSV_HEADER)
        for r in rows:
            writer.writerow([r['id'], r['date'], r['amount'], r['category'], r['description']])
    print(f"Exported {len(rows)} rows to {out_file}")


def plot_by_category(start_date=None, end_date=None):
    try:
        import matplotlib.pyplot as plt
    except Exception:
        print("matplotlib is required for plotting. Install it with: pip install matplotlib")
        return

    rows = read_all()
    if start_date:
        rows = [r for r in rows if r['date'] >= start_date]
    if end_date:
        rows = [r for r in rows if r['date'] <= end_date]
    if not rows:
        print("No data to plot.")
        return

    per_category = defaultdict(float)
    for r in rows:
        per_category[r['category']] += float(r['amount'])

    categories = list(per_category.keys())
    amounts = [per_category[c] for c in categories]

    plt.figure(figsize=(8, 6))
    plt.pie(amounts, labels=categories, autopct='%1.1f%%', startangle=140)
    plt.title('Spending by Category')
    plt.axis('equal')
    plt.show()


def plot_over_time():
    try:
        import matplotlib.pyplot as plt
    except Exception:
        print("matplotlib is required for plotting. Install it with: pip install matplotlib")
        return

    rows = read_all()
    if not rows:
        print("No data to plot.")
        return

    # Aggregate total per day
    per_day = defaultdict(float)
    for r in rows:
        per_day[r['date']] += float(r['amount'])

    dates = sorted(per_day.keys())
    totals = [per_day[d] for d in dates]

    x = [datetime.strptime(d, "%Y-%m-%d") for d in dates]
    plt.figure(figsize=(10, 5))
    plt.plot(x, totals)
    plt.xlabel('Date')
    plt.ylabel('Amount')
    plt.title('Spending Over Time')
    plt.tight_layout()
    plt.show()


def parse_args():
    p = argparse.ArgumentParser(description='Personal Expense Tracker')
    sub = p.add_subparsers(dest='cmd')

    p_add = sub.add_parser('add')
    p_add.add_argument('--amount', required=True)
    p_add.add_argument('--category', required=True)
    p_add.add_argument('--description', default='')
    p_add.add_argument('--date', default=None, help='YYYY-MM-DD')

    p_list = sub.add_parser('list')
    p_list.add_argument('--limit', type=int, default=20)
    p_list.add_argument('--category', default=None)
    p_list.add_argument('--start', default=None)
    p_list.add_argument('--end', default=None)

    p_report = sub.add_parser('report')
    p_report.add_argument('--month', default=None, help='YYYY-MM')
    p_report.add_argument('--start', default=None)
    p_report.add_argument('--end', default=None)

    p_export = sub.add_parser('export')
    p_export.add_argument('--out', required=True)
    p_export.add_argument('--category', default=None)
    p_export.add_argument('--start', default=None)
    p_export.add_argument('--end', default=None)

    p_plot = sub.add_parser('plot')
    p_plot.add_argument('--type', choices=['category', 'time'], required=True)
    p_plot.add_argument('--start', default=None)
    p_plot.add_argument('--end', default=None)

    return p.parse_args()


def main():
    args = parse_args()
    if args.cmd == 'add':
        add_expense(args.amount, args.category, args.description, args.date)
    elif args.cmd == 'list':
        list_expenses(limit=args.limit, category=args.category, start_date=args.start, end_date=args.end)
    elif args.cmd == 'report':
        if args.month:
            monthly_summary(args.month)
        else:
            report_range(start_date=args.start, end_date=args.end)
    elif args.cmd == 'export':
        export_csv(args.out, start_date=args.start, end_date=args.end, category=args.category)
    elif args.cmd == 'plot':
        if args.type == 'category':
            plot_by_category(start_date=args.start, end_date=args.end)
        else:
            plot_over_time()
    else:
        print("No command provided. Use -h for help.")


if __name__ == '__main__':
    main()
```

---

# Notes

* The files above are provided: `read.md` (the project script) and `project.py` (the full Python script).
* To upload to GitHub, follow the commands provided in the chat below.
