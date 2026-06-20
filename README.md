# Financial-Modeling
Grant Schwass Financial Modeling CPA
"""
Simple Financial Reporting Model
- Generates Income Statement, Balance Sheet, Cash Flow Statement, and ratios for N periods.
- No external dependencies. Paste into a file and run: python financial_model.py
- Edit the assumptions in `if __name__ == "__main__":` for different scenarios.
"""

from typing import List, Dict
import math

def format_money(x: float) -> str:
    return f"${x:,.0f}"

def build_income_statement(
    start_revenue: float,
    growth_rates: List[float],
    cogs_pct: float,
    opex_pct: float,
    d_and_a_pct: float,
    tax_rate: float,
) -> List[Dict]:
    periods = len(growth_rates)
    rev = start_revenue
    rows = []
    for i in range(periods):
        g = growth_rates[i]
        revenue = rev * (1 + g) if i > 0 else rev
        cogs = revenue * cogs_pct
        gross_profit = revenue - cogs
        opex = revenue * opex_pct
        ebitda = gross_profit - opex
        d_and_a = revenue * d_and_a_pct
        ebit = ebitda - d_and_a
        tax = max(0.0, ebit) * tax_rate
        net_income = ebit - tax
        rows.append({
            "period": i + 1,
            "revenue": revenue,
            "cogs": cogs,
            "gross_profit": gross_profit,
            "opex": opex,
            "ebitda": ebitda,
            "d_and_a": d_and_a,
            "ebit": ebit,
            "tax": tax,
            "net_income": net_income,
        })
        rev = revenue
    return rows

def build_balance_sheet(
    income_rows: List[Dict],
    start_cash: float,
    receivables_pct: float,
    inventory_pct: float,
    start_fixed_assets: float,
    capex_pct: float,
):
    bs_rows = []
    cash = start_cash
    fixed_assets = start_fixed_assets
    retained_earnings = 0.0

    prev_receivables = 0.0
    prev_inventory = 0.0

    for row in income_rows:
        revenue = row["revenue"]
        net_income = row["net_income"]
        # Working capital
        receivables = revenue * receivables_pct
        inventory = revenue * inventory_pct
        change_wc = (receivables - prev_receivables) + (inventory - prev_inventory)

        # Capex as % of revenue
        capex = - revenue * capex_pct  # negative cash flow
        # Depreciation (D&A) is in income statement as positive number
        d_and_a = row["d_and_a"]

        # Update fixed assets: add capex (positive addition) and subtract depreciation
        fixed_assets = fixed_assets + (-capex) - d_and_a

        # Cash: previous cash + net income + non-cash expense (d&a) + capex + change in WC
        # (note: capex is negative in our sign convention above)
        cash = cash + net_income + d_and_a + capex - change_wc

        retained_earnings += net_income

        current_assets = cash + receivables + inventory
        total_assets = current_assets + fixed_assets

        # Simple liabilities: assume short-term payables = 10% of revenue
        payables = revenue * 0.10
        equity = total_assets - payables

        bs_rows.append({
            "period": row["period"],
            "cash": cash,
            "receivables": receivables,
            "inventory": inventory,
            "current_assets": current_assets,
            "fixed_assets": fixed_assets,
            "total_assets": total_assets,
            "payables": payables,
            "equity": equity,
            "retained_earnings": retained_earnings,
            "capex": -capex,  # positive number for CAPEX spent
            "change_in_wc": change_wc,
        })

        prev_receivables = receivables
        prev_inventory = inventory

    return bs_rows

def build_cash_flow_statement(income_rows: List[Dict], bs_rows: List[Dict]) -> List[Dict]:
    cf_rows = []
    for inc, bs in zip(income_rows, bs_rows):
        net_income = inc["net_income"]
        d_and_a = inc["d_and_a"]
        capex = bs["capex"]  # positive value of capex spent
        change_in_wc = bs["change_in_wc"]
        # Operating CF (approx): net income + d&a - change in WC
        operating = net_income + d_and_a - change_in_wc
        investing = -capex
        financing = 0.0  # simple model: no new debt/equity
        net_change = operating + investing + financing

        cf_rows.append({
            "period": inc["period"],
            "operating_cf": operating,
            "investing_cf": investing,
            "financing_cf": financing,
            "net_change_in_cash": net_change,
        })
    return cf_rows

def compute_ratios(income_rows: List[Dict], bs_rows: List[Dict]) -> List[Dict]:
    r = []
    for inc, bs in zip(income_rows, bs_rows):
        revenue = inc["revenue"]
        gross_margin = inc["gross_profit"] / revenue if revenue else math.nan
        op_margin = inc["ebitda"] / revenue if revenue else math.nan
        net_margin = inc["net_income"] / revenue if revenue else math.nan
        current_ratio = bs["current_assets"] / bs["payables"] if bs["payables"] else math.nan
        roe = inc["net_income"] / bs["equity"] if bs["equity"] else math.nan

        r.append({
            "period": inc["period"],
            "gross_margin": gross_margin,
            "operating_margin": op_margin,
            "net_margin": net_margin,
            "current_ratio": current_ratio,
            "roe": roe,
        })
    return r

def print_table(title: str, rows: List[Dict], keys: List[str], fmt=None):
    print("\n" + title)
    # header
    header = ["Period"] + [k.replace("_", " ").title() for k in keys]
    print(" | ".join(header))
    print("-" * (len(" | ".join(header)) + 5))
    for row in rows:
        vals = [str(row["period"])]
        for k in keys:
            v = row.get(k, "")
            if fmt and k in fmt:
                vals.append(fmt[k](v))
            else:
                # fallback numeric formatting
                if isinstance(v, float):
                    vals.append(format_money(v))
                else:
                    vals.append(str(v))
        print(" | ".join(vals))

if __name__ == "__main__":
    # --- Assumptions (edit here) ---
    start_revenue = 1_000_000.0
    growth_rates = [0.05, 0.08, 0.06, 0.05, 0.04]  # 5 periods (year-over-year)
    cogs_pct = 0.60
    opex_pct = 0.18
    d_and_a_pct = 0.03
    tax_rate = 0.21

    start_cash = 100_000.0
    receivables_pct = 0.10
    inventory_pct = 0.08
    start_fixed_assets = 200_000.0
    capex_pct = 0.04  # capex as % of revenue each period

    # --- Build statements ---
    income = build_income_statement(start_revenue, growth_rates, cogs_pct, opex_pct, d_and_a_pct, tax_rate)
    bs = build_balance_sheet(income, start_cash, receivables_pct, inventory_pct, start_fixed_assets, capex_pct)
    cf = build_cash_flow_statement(income, bs)
    ratios = compute_ratios(income, bs)

    # --- Print human-friendly tables ---
    print_table("Income Statement (Simple)", income,
                ["revenue", "cogs", "gross_profit", "opex", "ebitda", "d_and_a", "ebit", "tax", "net_income"],
                fmt={
                    "revenue": format_money, "cogs": format_money, "gross_profit": format_money,
                    "opex": format_money, "ebitda": format_money, "d_and_a": format_money,
                    "ebit": format_money, "tax": format_money, "net_income": format_money,
                })

    print_table("Balance Sheet (Simple)", bs,
                ["cash", "receivables", "inventory", "current_assets", "fixed_assets", "total_assets", "payables", "equity", "retained_earnings", "capex"],
                fmt={k: format_money for k in ["cash", "receivables", "inventory", "current_assets", "fixed_assets", "total_assets", "payables", "equity", "retained_earnings", "capex"]})

    print_table("Cash Flow Statement (Simple)", cf,
                ["operating_cf", "investing_cf", "financing_cf", "net_change_in_cash"],
                fmt={k: format_money for k in ["operating_cf", "investing_cf", "financing_cf", "net_change_in_cash"]})

    # Ratios: format as percentages where relevant
    def pct(x):
        return f"{x*100:,.1f}%" if not math.isnan(x) else "N/A"

    print("\nRatios")
    print("Period | Gross Margin | Operating Margin | Net Margin | Current Ratio | ROE")
    print("-" * 80)
    for r in ratios:
        print(f"{r['period']} | {pct(r['gross_margin'])} | {pct(r['operating_margin'])} | {pct(r['net_margin'])} | {r['current_ratio']:.2f} | {pct(r['roe'])}")
