import csv
import argparse
from datetime import datetime, date
from pathlib import Path
from collections import defaultdict, Counter
import statistics


CSV_FILE = Path("expenses.csv")
CSV_HEADER = ["id", "date", "amount", "category", "description"]

def ensure_csv():
   
    if not CSV_FILE.exists():
        with CSV_FILE.open("w", newline='', encoding="utf-8") as f:
            writer = csv.writer(f)
            writer.writerow(CSV_HEADER)


def generate_id():
  
    ensure_csv()
    with CSV_FILE.open("r", newline='', encoding="utf-8") as f:
        reader = csv.DictReader(f)
        ids = [int(row["id"]) for row in reader if row["id"].isdigit()]
    return max(ids) + 1 if ids else 1



def add_expense(amount, category, description, date_str=None):

    ensure_csv()
    try:
        amt = float(amount)
    except ValueError:
        print("Amount must be a valid number.")
        return

    if date_str:
        try:
            d = datetime.strptime(date_str, "%Y-%m-%d").date()
        except ValueError:
            print("ate must be in YYYY-MM-DD format.")
            return
    else:
        d = date.today()

    rec_id = generate_id()
    with CSV_FILE.open("a", newline='', encoding="utf-8") as f:
        writer = csv.writer(f)
        writer.writerow([rec_id, d.isoformat(), f"{amt:.2f}", category.strip(), description.strip()])

    print(f"✅ Added expense ID={rec_id}: {amt:.2f} on {d} ({category})")


def read_all():

    ensure_csv()
    with CSV_FILE.open("r", newline='', encoding="utf-8") as f:
        return list(csv.DictReader(f))


def list_expenses(limit=None, category=None, start_date=None, end_date=None):

    rows = read_all()

    if category:
        rows = [r for r in rows if r["category"].lower() == category.lower()]
    if start_date:
        rows = [r for r in rows if r["date"] >= start_date]
    if end_date:
        rows = [r for r in rows if r["date"] <= end_date]

    rows = sorted(rows, key=lambda r: r["date"], reverse=True)
    if limit:
        rows = rows[:limit]

    if not rows:
        print("No expenses found.")
        return

    print(f"Showing {len(rows)} expenses:\n")
    for r in rows:
        print(f"{r['id']:>4} | {r['date']} | ₹{r['amount']:>8} | {r['category']:<12} | {r['description']}")


def monthly_summary(year_month):
  
    rows = read_all()
    filtered = [r for r in rows if r["date"].startswith(year_month)]
    if not filtered:
        print(f"No expenses for {year_month}.")
        return

    amounts = [float(r["amount"]) for r in filtered]
    total = sum(amounts)
    avg = statistics.mean(amounts)
    per_category = defaultdict(float)

    for r in filtered:
        per_category[r["category"]] += float(r["amount"])

    print(f" Summary for {year_month}")
    print(f"  Total spent: ₹{total:.2f}")
    print(f"  Transactions: {len(amounts)}")
    print(f"  Average per transaction: ₹{avg:.2f}\n")
    print("  Breakdown by category:")
    for cat, amt in sorted(per_category.items(), key=lambda x: x[1], reverse=True):
        pct = (amt / total) * 100 if total else 0
        print(f"    {cat:<12} ₹{amt:>8.2f} ({pct:>5.1f}%)")


def report_range(start_date=None, end_date=None):

    rows = read_all()
    if start_date:
        rows = [r for r in rows if r["date"] >= start_date]
    if end_date:
        rows = [r for r in rows if r["date"] <= end_date]

    if not rows:
        print("No expenses fo
