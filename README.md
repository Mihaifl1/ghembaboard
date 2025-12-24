import mysql.connector
import tkinter as tk
from tkinter import ttk, messagebox
from datetime import datetime, timedelta, date
import calendar
import matplotlib.pyplot as plt
from matplotlib.backends.backend_tkagg import FigureCanvasTkAgg

# ================= DB CONFIG =================
DB_CONFIG = {
    "host": "192.168.66.248",
    "user": "root",
    "password": "M9vdf!gdcf!hdf9",
    "database": "validations",
    "charset": "utf8mb4",
    "use_unicode": True
}

# ====== CONFIG TURE (exact ca Excelul tău) ======
SHIFT_HOURS = {1: 8, 2: 5, 3: 8}  # Turul I=8h, Turul II=5h, Turul III=8h
SHIFTS_TABLE = "device_type_shifts"  # tabel ture pe tip
HOLIDAYS_TABLE = "calendar_holidays"  # opțional (col: date DATE)

REFRESH_MS = 60000  # 60 sec

# ================= TARGETE (pagina 1) =================
TARGET_MTTR_OK = 30
TARGET_MTTR_WARN = 45
TARGET_DOWNTIME_OK = 30
TARGET_DOWNTIME_WARN = 60
TARGET_PM_OK = 95
TARGET_PM_WARN = 90
TARGET_MTBF_OK = 120
TARGET_MTBF_WARN = 80

# ================= PM ORDINARY CONFIG =================
MAINT_PLAN_TABLE = "maintenance_plan"
ORD_INTERVAL_WORKDAYS = {
    "daily": 1,
    "weekly": 5,
    "monthly": 22,
    "three_month": 66,
    "six_month": 132,
    "yearly": 264
}
WEEKDAY_MAP = {1: 0, 2: 1, 3: 2, 4: 3, 5: 4}


def subtract_workdays(end_date: date, workdays: int, holidays: set = None) -> date:
    """Scade N zile lucrătoare dintr-o dată."""
    if holidays is None:
        holidays = set()
    d = end_date
    left = int(workdays)
    while left > 0:
        d -= timedelta(days=1)
        if d.weekday() < 5 and d not in holidays:
            left -= 1
    return d


# ================= HELPERS =================
def color_by_value(val, ok, warn, reverse=False):
    if reverse:
        if val <= ok:
            return "#1b5e20"
        if val <= warn:
            return "#f9a825"
        return "#b71c1c"
    else:
        if val >= ok:
            return "#1b5e20"
        if val >= warn:
            return "#f9a825"
        return "#b71c1c"


def sec_to_min(sec):
    return int(round(sec / 60)) if sec else 0


def week_range_from_iso(year, week):
    start = datetime.fromisocalendar(year, week, 1)
    end = start + timedelta(days=6, hours=23, minutes=59, seconds=59)
    return start, end


def table_exists(conn, table_name: str) -> bool:
    q = """
        SELECT COUNT(*) AS cnt
        FROM information_schema.TABLES
        WHERE TABLE_SCHEMA=%s AND TABLE_NAME=%s
    """
    cur = conn.cursor(dictionary=True)
    cur.execute(q, (DB_CONFIG["database"], table_name))
    r = cur.fetchone()
    cur.close()
    return bool(r and r.get("cnt", 0) > 0)


def ensure_shifts_table(conn):
    cur = conn.cursor()
    cur.execute(f"""
        CREATE TABLE IF NOT EXISTS `{SHIFTS_TABLE}` (
            `id_type` INT NOT NULL,
            `shifts` TINYINT NOT NULL DEFAULT 2,
            `updated_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
            PRIMARY KEY (`id_type`)
        ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
    """)
    conn.commit()
    cur.close()


def get_holidays_set(conn, year: int):
    if not table_exists(conn, HOLIDAYS_TABLE):
        return set()
    cur = conn.cursor(dictionary=True)
    cur.execute(f"SELECT `date` FROM `{HOLIDAYS_TABLE}` WHERE YEAR(`date`)=%s", (year,))
    rows = cur.fetchall()
    cur.close()
    out = set()
    for r in rows:
        if r.get("date"):
            out.add(r["date"])
    return out


def workdays_in_month(year: int, month: int, holidays: set):
    _, last_day = calendar.monthrange(year, month)
    cnt = 0
    for d in range(1, last_day + 1):
        dt = date(year, month, d)
        if dt.weekday() < 5 and dt not in holidays:
            cnt += 1
    return cnt



def hours_per_day_by_shifts(shifts: int) -> int:
    shifts = max(1, min(3, int(shifts)))
    return sum(SHIFT_HOURS.get(s, 0) for s in range(1, shifts + 1))


def workdays_between(d1: date, d2: date, holidays: set = None) -> int:
    """Număr zile lucrătoare între două date (d1 -> d2). d2 exclusiv."""
    if holidays is None:
        holidays = set()
    if d1 is None or d2 is None:
        return 0
    if d1 > d2:
        d1, d2 = d2, d1

    cnt = 0
    cur = d1
    while cur < d2:
        if cur.weekday() < 5 and cur not in holidays:
            cnt += 1
        cur += timedelta(days=1)
    return cnt


def status_by_pct(pct: float):
    if pct >= TARGET_PM_OK:
        return "VERDE", "#1b5e20"
    if pct >= TARGET_PM_WARN:
        return "GALBEN", "#f9a825"
    return "ROȘU", "#b71c1c"


# ================= PM LOGIC HELPERS =================
CALENDAR_LIMITS = {
    "weekly": 5,
    "monthly": 22,
    "three months": 66,
    "three_month": 66,
    "six months": 132,
    "six_month": 132,
    "yearly": 264
}


# praguri example pentru number of shuts (le poți regla)


def status_calendar_only(wdays: int, limit: int) -> str:
    if wdays <= int(limit * 0.8):
        return "VERDE"
    if wdays <= limit:
        return "GALBEN"
    return "ROȘU"


# ================= APP =================
class GembaBoardApp:
    def __init__(self, root):
        self.root = root
        self.root.title("Gemba Board")
        self.root.configure(bg="#111")
        self.root.attributes("-fullscreen", True)

        today = datetime.now()
        self.current_year = today.year
        self.current_week = today.isocalendar()[1]

        # ---------- HEADER ----------
        header = tk.Frame(root, bg="#222")
        header.pack(fill="x")

        tk.Label(
            header, text="🏭 GEMBA BOARD – MENTENANȚĂ",
            font=("Arial", 30, "bold"), fg="white", bg="#222"
        ).pack(side="left", padx=20)

        self.time_lbl = tk.Label(header, font=("Arial", 22), fg="white", bg="#222")
        self.time_lbl.pack(side="right", padx=20)

        # ---------- WEEK NAV ----------
        nav = tk.Frame(root, bg="#111")
        nav.pack(fill="x")

        tk.Button(nav, text="◀", font=("Arial", 18), command=self.prev_week).pack(side="left", padx=10)
        self.week_lbl = tk.Label(nav, font=("Arial", 22, "bold"), fg="white", bg="#111")
        self.week_lbl.pack(side="left", padx=20)
        tk.Button(nav, text="▶", font=("Arial", 18), command=self.next_week).pack(side="left", padx=10)
        tk.Button(nav, text="⟳ Azi", font=("Arial", 14), command=self.reset_week).pack(side="left", padx=20)

        # ---------- PERIOD SELECTOR ----------
        self.period_var = tk.StringVar(value="Săptămână")

        self.period_cb = ttk.Combobox(
            nav,
            textvariable=self.period_var,
            values=["Săptămână", "Lună curentă", "Lună anterioară", "Interval custom"],
            state="readonly",
            width=18
        )
        self.period_cb.pack(side="left", padx=15)
        self.period_cb.bind("<<ComboboxSelected>>", lambda e: self.load_gemba_page1())

        self.date_from = tk.Entry(nav, width=10)
        self.date_to = tk.Entry(nav, width=10)

        self.date_from.insert(0, datetime.now().replace(day=1).strftime("%Y-%m-%d"))
        self.date_to.insert(0, datetime.now().strftime("%Y-%m-%d"))

        self.date_from.pack(side="left", padx=5)
        self.date_to.pack(side="left", padx=5)

        tk.Button(nav, text="Aplică", font=("Arial", 12), command=self.load_gemba_page1).pack(side="left", padx=10)

        tk.Label(nav, text="Locație:", font=("Arial", 14), fg="white", bg="#111") \
            .pack(side="left", padx=(20, 5))

        self.location_var = tk.StringVar(value="ALL")

        self.location_cb = ttk.Combobox(
            nav,
            textvariable=self.location_var,
            state="readonly",
            width=8,
            values=["ALL", "FL", "SG"]
        )
        self.location_cb.pack(side="left")
        self.location_cb.bind("<<ComboboxSelected>>", lambda e: self.refresh())

        # ---------- PAGE NAV ----------
        page_nav = tk.Frame(root, bg="#111")
        page_nav.pack(fill="x", pady=5)

        tk.Button(page_nav, text="📊 GEMBA", font=("Arial", 14),
                  command=lambda: self.show_page(1)).pack(side="left", padx=10)
        tk.Button(page_nav, text="🏭 DISPOZITIVE", font=("Arial", 14),
                  command=lambda: self.show_page(2)).pack(side="left", padx=10)
        tk.Button(page_nav, text="🗓 ORE/AN", font=("Arial", 14),
                  command=lambda: self.show_page(3)).pack(side="left", padx=10)

        # ---------- PAGES ----------
        self.page1 = tk.Frame(root, bg="#111")
        self.page2 = tk.Frame(root, bg="#111")
        self.page3 = tk.Frame(root, bg="#111")
        self.page1.pack(fill="both", expand=True)

        # ================= PAGE 1 =================
        self.kpi_frame = tk.Frame(self.page1, bg="#111")
        self.kpi_frame.pack(fill="x", padx=10, pady=10)

        self.box_mtbf = self._kpi_box("MTBF")
        self.box_mttr = self._kpi_box("MTTR")
        self.box_dt = self._kpi_box("DOWNTIME")
        self.box_bd = self._kpi_box("BREAKDOWNS")
        self.box_pm = self._kpi_box("PM %")

        self.mid_frame = tk.Frame(self.page1, bg="#111")
        self.mid_frame.pack(fill="x", padx=20, pady=10)

        tk.Label(
            self.mid_frame,
            text="MENTENANȚĂ EXTRAORDINARĂ PE TIP DISPOZITIV",
            font=("Arial", 22, "bold"), fg="white", bg="#111"
        ).pack(anchor="w", pady=(0, 10))

        self.mid_list = tk.Frame(self.mid_frame, bg="#111")
        self.mid_list.pack(fill="x")

        # --------- PM ORDINARY ----------
        self.pm_frame = tk.Frame(self.page1, bg="#111")
        self.pm_frame.pack(fill="both", expand=True, padx=20, pady=(10, 10))

        pm_header = tk.Frame(self.pm_frame, bg="#111")
        pm_header.pack(fill="x", pady=(0, 6))

        tk.Label(
            pm_header,
            text="MENTENANȚĂ ORDINARĂ – STATUS LA ZI (PLAN vs REALIZAT)",
            font=("Arial", 22, "bold"), fg="white", bg="#111"
        ).pack(side="left", anchor="w")

        self.pm_counts_lbl = tk.Label(
            pm_header,
            text="VERDE: 0   GALBEN: 0   ROȘU: 0",
            font=("Arial", 16, "bold"), fg="white", bg="#111"
        )
        self.pm_counts_lbl.pack(side="right")
        # ===== FILTRU PERIODICITATE PM =====
        pm_filter_frame = tk.Frame(self.pm_frame, bg="#111")
        pm_filter_frame.pack(fill="x", pady=(0, 6))

        tk.Label(
            pm_filter_frame,
            text="Periodicitate PM:",
            font=("Arial", 14),
            fg="white",
            bg="#111"
        ).pack(side="left", padx=(0, 10))

        self.pm_period_var = tk.StringVar(value="weekly")  # DEFAULT

        self.pm_period_cb = ttk.Combobox(
            pm_filter_frame,
            textvariable=self.pm_period_var,
            state="readonly",
            width=16,
            values=[
                "weekly",
                "monthly",
                "three_month",
                "six_month",
                "yearly",
                "all"
            ]
        )
        self.pm_period_cb.pack(side="left")
        self.pm_period_cb.bind(
            "<<ComboboxSelected>>",
            lambda e: self.load_pm_ordinary_status(datetime.now())
        )

        # ===== TABEL PM =====
        pm_table_frame = tk.Frame(self.pm_frame, bg="#111")
        pm_table_frame.pack(fill="both", expand=True)

        cols_pm = ("device_type", "production", "period", "planned", "done", "pct", "last", "wdays", "status")
        self.pm_table = ttk.Treeview(pm_table_frame, columns=cols_pm, show="headings")

        self.pm_table.heading("device_type", text="Tip dispozitiv")
        self.pm_table.heading("production", text="În prod.")
        self.pm_table.heading("period", text="Periodicitate")
        self.pm_table.heading("planned", text="Plan")
        self.pm_table.heading("done", text="Realizat")
        self.pm_table.heading("pct", text="%")
        self.pm_table.heading("last", text="Ultima PM")
        self.pm_table.heading("wdays", text="Zile L-V")
        self.pm_table.heading("status", text="Status")

        self.pm_table.column("device_type", width=360, anchor="w")
        self.pm_table.column("production", width=90, anchor="center")
        self.pm_table.column("period", width=130, anchor="center")
        self.pm_table.column("planned", width=90, anchor="e")
        self.pm_table.column("done", width=90, anchor="e")
        self.pm_table.column("pct", width=80, anchor="e")
        self.pm_table.column("last", width=120, anchor="center")
        self.pm_table.column("wdays", width=90, anchor="e")
        self.pm_table.column("status", width=90, anchor="center")

        self.pm_table.pack(fill="both", expand=True)

        sb_pm = ttk.Scrollbar(pm_table_frame, orient="vertical", command=self.pm_table.yview)
        self.pm_table.configure(yscrollcommand=sb_pm.set)
        sb_pm.pack(side="right", fill="y")

        self.pm_table.tag_configure("VERDE", foreground="#1b5e20")
        self.pm_table.tag_configure("GALBEN", foreground="#f9a825")
        self.pm_table.tag_configure("ROȘU", foreground="#b71c1c")

        # ================= PAGE 2 =================
        tk.Label(
            self.page2,
            text="🏭 DISPOZITIVE ÎN PRODUCȚIE (PE TIP) – dublu click pe TURE",
            font=("Arial", 24, "bold"), fg="white", bg="#111"
        ).pack(anchor="w", padx=20, pady=(20, 10))

        table_frame2 = tk.Frame(self.page2, bg="#111")
        table_frame2.pack(fill="both", expand=True, padx=20, pady=10)

        cols2 = ("device_type", "production", "shifts")
        self.devices_table = ttk.Treeview(table_frame2, columns=cols2, show="headings", height=20)

        self.devices_table.heading("device_type", text="Tip dispozitiv")
        self.devices_table.heading("production", text="În producție")
        self.devices_table.heading("shifts", text="Ture (1/2/3)")

        self.devices_table.column("device_type", width=520, anchor="w")
        self.devices_table.column("production", width=160, anchor="center")
        self.devices_table.column("shifts", width=160, anchor="center")

        self.devices_table.pack(fill="both", expand=True)

        sb2 = ttk.Scrollbar(table_frame2, orient="vertical", command=self.devices_table.yview)
        self.devices_table.configure(yscrollcommand=sb2.set)
        sb2.pack(side="right", fill="y")

        self._setup_tree_style()
        self.devices_table.tag_configure("even", background="#ffffff")
        self.devices_table.tag_configure("odd", background="#f2f2f2")

        self._shift_editor = None
        self.devices_table.bind("<Double-1>", self._on_double_click_devices)

        # ================= PAGE 3 =================
        top3 = tk.Frame(self.page3, bg="#111")
        top3.pack(fill="x", padx=20, pady=(20, 8))

        tk.Label(top3, text="🗓 ORE LUCRATE PE AN", font=("Arial", 24, "bold"),
                 fg="white", bg="#111").pack(side="left")

        self.year_var = tk.IntVar(value=datetime.now().year)
        year_spin = tk.Spinbox(top3, from_=2020, to=2050, width=6, font=("Arial", 16),
                               textvariable=self.year_var, command=self.load_year_hours_page)
        year_spin.pack(side="left", padx=20)

        tk.Button(top3, text="Reîncarcă", font=("Arial", 12),
                  command=self.load_year_hours_page).pack(side="left")

        self.leave_frame = tk.Frame(self.page3, bg="#111")
        self.leave_frame.pack(fill="x", padx=20, pady=10)

        table_frame3 = tk.Frame(self.page3, bg="#111")
        table_frame3.pack(fill="both", expand=True, padx=20, pady=10)

        months = ["Jan", "Feb", "Mar", "Apr", "May", "Jun", "Jul", "Aug", "Sep", "Oct", "Nov", "Dec"]
        cols3 = ("device_type", "number", "shifts") + tuple(months) + ("total",)
        self.year_table = ttk.Treeview(table_frame3, columns=cols3, show="headings", height=18)

        self.year_table.heading("device_type", text="Tip dispozitiv")
        self.year_table.heading("number", text="Number")
        self.year_table.heading("shifts", text="Ture")
        for m in months:
            self.year_table.heading(m, text=m)
        self.year_table.heading("total", text="Total")

        self.year_table.column("device_type", width=300, anchor="w")
        self.year_table.column("number", width=90, anchor="center")
        self.year_table.column("shifts", width=70, anchor="center")
        for m in months:
            self.year_table.column(m, width=90, anchor="e")
        self.year_table.column("total", width=110, anchor="e")

        self.year_table.pack(fill="both", expand=True)

        sb3 = ttk.Scrollbar(table_frame3, orient="vertical", command=self.year_table.yview)
        self.year_table.configure(yscrollcommand=sb3.set)
        sb3.pack(side="right", fill="y")

        self.year_table.tag_configure("even", background="#ffffff")
        self.year_table.tag_configure("odd", background="#f2f2f2")

        root.bind("<Escape>", lambda e: root.destroy())

        self.refresh()
        self.show_page(1)

    def location_sql_filter(self, alias="d"):
        """
        alias = aliasul tabelei devices (d)
        NULL => FL
        """
        loc = self.location_var.get()
        if loc == "ALL":
            return ""
        return f" AND COALESCE({alias}.location, 'FL') = '{loc}' "

    def show_extraordinary_details(self, device_type):
        period_start, period_end = self.get_period()

        win = tk.Toplevel(self.root)
        win.title(f"Intervenții EXTRAORDINARE – {device_type}")
        win.geometry("1600x650")
        win.configure(bg="#111")

        tk.Label(
            win,
            text=f"Intervenții EXTRAORDINARE – {device_type}",
            font=("Arial", 20, "bold"),
            fg="white",
            bg="#111"
        ).pack(anchor="w", padx=20, pady=10)

        frame = tk.Frame(win, bg="#111")
        frame.pack(fill="both", expand=True, padx=20, pady=10)

        cols = ("id", "created_at", "duration", "operator", "description", "type_machine_id", "type_ment_id")
        table = ttk.Treeview(frame, columns=cols, show="headings")
        table.pack(fill="both", expand=True)

        for c in cols:
            table.heading(c, text=c.upper())
            table.column(c, width=220, anchor="center")

        sb = ttk.Scrollbar(frame, orient="vertical", command=table.yview)
        table.configure(yscrollcommand=sb.set)
        sb.pack(side="right", fill="y")

        try:
            conn = mysql.connector.connect(**DB_CONFIG)
            cur = conn.cursor(dictionary=True)

            q = f"""
                SELECT
                    i.id,
                    i.created_at,
                    i.duration,
                    u.name        AS operator,
                    i.note        AS description,
                    i.id_type_machine    AS type_machine_id,
                    i.id_type_mentenance AS type_ment_id
                FROM interventions i
                JOIN device_types dt 
                    ON i.id_type_machine = dt.id
                JOIN type_mentenance tm 
                    ON i.id_type_mentenance = tm.id
                LEFT JOIN users u 
                    ON i.id_user = u.id
                WHERE tm.name LIKE '%Extraordinary%'
                  AND dt.name = %s
                  AND i.created_at BETWEEN %s AND %s
                  {self.location_sql_filter("i")}
                ORDER BY i.created_at DESC
            """

            cur.execute(q, (device_type, period_start, period_end))
            rows = cur.fetchall()

            for r in rows:
                table.insert(
                    "",
                    "end",
                    values=(
                        r["id"],
                        r["created_at"].strftime("%d.%m.%Y %H:%M") if r["created_at"] else "",
                        r["duration"],
                        r.get("operator", ""),
                        r.get("description", ""),
                        r.get("type_machine_id", ""),
                        r.get("type_ment_id", "")
                    )
                )

            cur.close()
            conn.close()

        except Exception as e:
            messagebox.showerror("Eroare DB", str(e))

            # ---------- PERIOD ----------
    def get_period(self):
        today = date.today()

        if self.period_var.get() == "Lună curentă":
            start = today.replace(day=1)
            end = today
        elif self.period_var.get() == "Lună anterioară":
            first = today.replace(day=1)
            end = first - timedelta(days=1)
            start = end.replace(day=1)
        elif self.period_var.get() == "Interval custom":
            try:
                start = datetime.strptime(self.date_from.get(), "%Y-%m-%d").date()
                end = datetime.strptime(self.date_to.get(), "%Y-%m-%d").date()
            except Exception:
                messagebox.showerror("Eroare", "Interval custom: format date corect este YYYY-MM-DD.")
                start = today.replace(day=1)
                end = today
        else:  # Săptămână
            ws, we = week_range_from_iso(self.current_year, self.current_week)
            start, end = ws.date(), we.date()

        return datetime.combine(start, datetime.min.time()), datetime.combine(end, datetime.max.time())

    # ---------- STYLE ----------
    def _setup_tree_style(self):
        style = ttk.Style()
        style.theme_use("default")
        style.configure("Treeview",
                        font=("Arial", 14),
                        rowheight=32,
                        background="white",
                        fieldbackground="white",
                        borderwidth=1,
                        relief="solid")
        style.configure("Treeview.Heading",
                        font=("Arial", 14, "bold"),
                        relief="solid",
                        borderwidth=1)
        style.map("Treeview", background=[("selected", "#cce6ff")])

    # ---------- UI HELPERS ----------
    def _kpi_box(self, title):
        f = tk.Frame(self.kpi_frame, bg="#444")
        f.pack(side="left", expand=True, fill="both", padx=6, pady=6)
        tk.Label(f, text=title, font=("Arial", 20, "bold"), fg="white", bg=f["bg"]).pack(pady=(10, 0))
        v = tk.Label(f, text="-", font=("Arial", 54, "bold"), fg="white", bg=f["bg"])
        v.pack(expand=True, pady=(0, 10))
        return {"frame": f, "value": v}

    def _set_box(self, box, text, color):
        box["frame"].configure(bg=color)
        for w in box["frame"].winfo_children():
            w.configure(bg=color)
        box["value"].configure(text=text)

    # ---------- PAGE SWITCH ----------
    def show_page(self, page):
        self.page1.pack_forget()
        self.page2.pack_forget()
        self.page3.pack_forget()

        if page == 1:
            self.page1.pack(fill="both", expand=True)
            self.load_gemba_page1()
        elif page == 2:
            self.page2.pack(fill="both", expand=True)
            self.load_devices_page()
        else:
            self.page3.pack(fill="both", expand=True)
            self.load_year_hours_page()

    # ---------- WEEK NAV ----------
    def prev_week(self):
        self.current_week -= 1
        if self.current_week < 1:
            self.current_year -= 1
            self.current_week = 52
        self.load_gemba_page1()

    def next_week(self):
        self.current_week += 1
        if self.current_week > 52:
            self.current_year += 1
            self.current_week = 1
        self.load_gemba_page1()

    def reset_week(self):
        today = datetime.now()
        self.current_year = today.year
        self.current_week = today.isocalendar()[1]
        self.load_gemba_page1()

    # ---------- REFRESH ----------
    def refresh(self):
        self.time_lbl.config(text=datetime.now().strftime("%H:%M  %d.%m.%Y"))
        if self.page1.winfo_ismapped():
            self.load_gemba_page1()
        self.root.after(REFRESH_MS, self.refresh)

    # ================= PAGE 1 LOAD =================
    def load_gemba_page1(self):
        week_start, _ = week_range_from_iso(self.current_year, self.current_week)
        period_start, period_end = self.get_period()
        self.week_lbl.config(text=f"{self.period_var.get()} ({period_start:%d.%m} – {period_end:%d.%m})")

        try:
            conn = mysql.connector.connect(**DB_CONFIG)
            cur = conn.cursor(dictionary=True)

            q_kpi = """
                SELECT
                    tm.name AS type_name,
                    COUNT(*) AS cnt,
                    COALESCE(SUM(TIME_TO_SEC(i.duration)),0) AS sec_total
                FROM interventions i
                JOIN type_mentenance tm ON i.id_type_mentenance = tm.id
                WHERE i.created_at BETWEEN %s AND %s
                GROUP BY tm.name;
            """
            cur.execute(q_kpi, (period_start, period_end))
            rows = cur.fetchall()

            total_cnt = total_sec = extraordinary_cnt = extraordinary_sec = pm_cnt = 0
            for r in rows:
                name = r.get("type_name") or ""
                cnt = int(r.get("cnt") or 0)
                sec = int(r.get("sec_total") or 0)

                total_cnt += cnt
                total_sec += sec

                if "Extraordinary" in name:
                    extraordinary_cnt += cnt
                    extraordinary_sec += sec
                if "Predictive" in name or "Ordinary" in name:
                    pm_cnt += cnt

            # ===== MTTR (doar extraordinary) =====
            mttr_min = round((extraordinary_sec / 60) / extraordinary_cnt, 1) if extraordinary_cnt else 0

            downtime_min = sec_to_min(extraordinary_sec)
            pm_pct = round((pm_cnt / total_cnt) * 100, 1) if total_cnt else 0

            # ===== PRODUCTION TIME =====
            ensure_shifts_table(conn)
            q_prod = f"""
                SELECT
                    dt.id,
                    COUNT(d.id) AS production,
                    COALESCE(s.shifts,2) AS shifts
                FROM devices d
                JOIN device_types dt ON d.id_type = dt.id
                LEFT JOIN `{SHIFTS_TABLE}` s ON s.id_type = dt.id
                WHERE d.status='Production'
                {self.location_sql_filter("d")}
                GROUP BY dt.id, s.shifts
            """

            cur.execute(q_prod)
            prod_rows = cur.fetchall()

            holidays = get_holidays_set(conn, period_start.year)
            workdays = workdays_between(period_start.date(), period_end.date() + timedelta(days=1), holidays)

            prod_hours = 0.0
            for r in prod_rows:
                prod = int(r.get("production") or 0)
                hpd = hours_per_day_by_shifts(int(r.get("shifts") or 2))
                prod_hours += prod * workdays * hpd

            downtime_hours = extraordinary_sec / 3600.0

            if extraordinary_cnt > 0:
                mtbf_shopfloor = round(max(prod_hours - downtime_hours, 0) / extraordinary_cnt, 1)
                mtbf_text = f"{mtbf_shopfloor}h"
                mtbf_color = color_by_value(mtbf_shopfloor, TARGET_MTBF_OK, TARGET_MTBF_WARN)
            else:
                mtbf_shopfloor = float("inf")  # 🔹 DEFINIT
                mtbf_text = "∞"
                mtbf_color = "#1b5e20"

            self._set_box(
                self.box_mtbf,
                mtbf_text,
                mtbf_color
            )
            self._set_box(
                self.box_mttr,
                f"{mttr_min}m",
                color_by_value(mttr_min, TARGET_MTTR_OK, TARGET_MTTR_WARN, reverse=True)
            )

            self._set_box(self.box_dt, f"{downtime_min}m",
                          color_by_value(downtime_min, TARGET_DOWNTIME_OK, TARGET_DOWNTIME_WARN, True))
            self._set_box(self.box_bd, f"{extraordinary_cnt}",
                          color_by_value(extraordinary_cnt, 0, 1, True))
            self._set_box(self.box_pm, f"{pm_pct}%",
                          color_by_value(pm_pct, TARGET_PM_OK, TARGET_PM_WARN))


            # LIST – extraordinary by device type
            for w in self.mid_list.winfo_children():
                w.destroy()

            q_mid = """
                SELECT
                    dt.name AS device_type,
                    COUNT(*) AS cnt,
                    COALESCE(SUM(TIME_TO_SEC(i.duration)),0) AS sec_total
                FROM interventions i
                JOIN device_types dt ON i.id_type_machine = dt.id
                JOIN type_mentenance tm ON i.id_type_mentenance = tm.id
                WHERE tm.name LIKE '%Extraordinary%'
                  AND i.created_at BETWEEN %s AND %s
                GROUP BY dt.name
                ORDER BY sec_total DESC;
            """
            cur.execute(q_mid, (period_start, period_end))
            mids = cur.fetchall()

            for m in mids:
                device_type = (m.get("device_type") or "").strip()
                cnt = int(m.get("cnt") or 0)
                minutes = sec_to_min(int(m.get("sec_total") or 0))

                lbl = tk.Label(
                    self.mid_list,
                    text=f"{device_type:24} | {cnt} intervenții | {minutes} min",
                    font=("Arial", 18),
                    fg="white",
                    bg="#111",
                    cursor="hand2"  # 🔹 arată că e clickabil
                )
                lbl.pack(anchor="w", pady=2)

                # 🔵 DUBLU CLICK
                lbl.bind(
                    "<Double-Button-1>",
                    lambda e, dev=device_type: self.show_extraordinary_details(dev)
                )

            cur.close()
            conn.close()

            # PM Ordinary rămâne pe săptămâna curentă (la zi)
            self.load_pm_ordinary_status(week_start)

        except Exception as e:
            print("DB ERROR PAGE1:", e)

    def draw_bar_chart(self, data):
        for w in self.graph_frame.winfo_children():
            w.destroy()
        if not data:
            return

        labels = [d.get("machine_type") for d in data]
        values = [int(d.get("cnt") or 0) for d in data]

        fig, ax = plt.subplots(figsize=(9, 2))
        fig.patch.set_facecolor("#111")
        ax.set_facecolor("#111")

        ax.bar(labels, values)
        ax.set_title("Intervenții (toate tipurile)", color="white")

        ax.tick_params(axis='x', colors='white')
        ax.tick_params(axis='y', colors='white')
        for spine in ax.spines.values():
            spine.set_color('white')

        canvas = FigureCanvasTkAgg(fig, master=self.graph_frame)
        canvas.draw()
        canvas.get_tk_widget().pack(fill="x", expand=True)

    def load_pm_ordinary_status(self, week_start_dt: datetime):
        for r in self.pm_table.get_children():
            self.pm_table.delete(r)

        cnt_green = cnt_yellow = cnt_red = 0

        try:
            conn = mysql.connector.connect(**DB_CONFIG)
            cur = conn.cursor(dictionary=True)

            # ===== Dispozitive în producție =====
            cur.execute(f"""
                SELECT
                    dt.id   AS id_type,
                    dt.name AS device_type,
                    COUNT(d.id) AS production
                FROM devices d
                JOIN device_types dt ON d.id_type = dt.id
                WHERE d.status = 'Production'
                {self.location_sql_filter("d")}
                GROUP BY dt.id, dt.name
            """)
            devices = cur.fetchall()

            today = date.today()
            holidays = get_holidays_set(conn, today.year)

            # ===== Planuri PM =====
            cur.execute("""
                SELECT
                    dt.name AS device_type,
                    mp.periodicity
                FROM maintenance_plan mp
                JOIN device_types dt ON mp.id_type_device = dt.id
                WHERE mp.periodicity IS NOT NULL
                  AND mp.periodicity <> ''
            """)
            plan_rows = cur.fetchall()

            # ===== ID-uri Ordinary =====
            cur.execute("SELECT id FROM type_mentenance WHERE name LIKE '%Ordinary%'")
            ord_ids = [str(x["id"]) for x in cur.fetchall()] or ["-1"]
            ord_ids_sql = ",".join(ord_ids)

            for dev in devices:
                dev_name = dev["device_type"]
                prod = int(dev["production"] or 0)

                periodicities = sorted({
                    (p.get("periodicity") or "").strip().lower()
                    for p in plan_rows
                    if (p.get("device_type") or "").strip().lower() == dev_name.strip().lower()
                })

                selected_period = self.pm_period_var.get()

                for periodicity in periodicities:
                    if selected_period != "all" and periodicity != selected_period:
                        continue

                    if periodicity in ("daily",) or "shut" in periodicity:
                        continue
                    limit = CALENDAR_LIMITS[periodicity]

                    # ===== Ultima PM =====
                    cur.execute(f"""
                        SELECT MAX(created_at) AS last_pm
                        FROM interventions
                        WHERE id_type_machine = %s
                          AND id_type_mentenance IN ({ord_ids_sql})
                    """, (dev["id_type"],))
                    last_dt = (cur.fetchone() or {}).get("last_pm")

                    if last_dt:
                        last_date = last_dt.date()
                        wdays = workdays_between(last_date, today + timedelta(days=1), holidays)
                        last_txt = last_date.strftime("%d.%m.%Y")
                    else:
                        wdays = limit
                        last_txt = "-"

                    # ===== PLAN =====
                    planned = prod

                    # ===== REALIZAT =====
                    window_start = subtract_workdays(today, limit, holidays)
                    cur.execute(f"""
                        SELECT COUNT(*) AS done_cnt
                        FROM interventions
                        WHERE id_type_machine = %s
                          AND id_type_mentenance IN ({ord_ids_sql})
                          AND created_at BETWEEN %s AND %s
                    """, (
                        dev["id_type"],
                        datetime.combine(window_start, datetime.min.time()),
                        datetime.combine(today, datetime.max.time())
                    ))
                    realizat = int((cur.fetchone() or {}).get("done_cnt") or 0)

                    # ===== PROCENT =====
                    pct = round((realizat / planned) * 100, 1) if planned > 0 else 0.0
                    pct = min(pct, 100.0)

                    # ===== STATUS =====
                    if pct >= TARGET_PM_OK:
                        status = "VERDE"
                        cnt_green += 1
                    elif pct >= TARGET_PM_WARN:
                        status = "GALBEN"
                        cnt_yellow += 1
                    else:
                        status = "ROȘU"
                        cnt_red += 1

                    period_label = {
                        "weekly": "Săptămânal",
                        "monthly": "Lunar",
                        "three_month": "Trimestrial",
                        "six_month": "Semestrial",
                        "yearly": "Anual"
                    }.get(periodicity, periodicity)

                    self.pm_table.insert(
                        "",
                        "end",
                        values=(
                            dev_name,
                            prod,
                            period_label,
                            planned,
                            realizat,
                            f"{pct}%",
                            last_txt,
                            wdays,
                            status
                        ),
                        tags=(status,)
                    )

            self.pm_counts_lbl.config(
                text=f"VERDE: {cnt_green}   GALBEN: {cnt_yellow}   ROȘU: {cnt_red}"
            )

            cur.close()
            conn.close()

        except Exception as e:
            print("DB ERROR PM ORDINARY:", e)

    # ================= PAGE 2 LOAD + EDIT SHIFTS =================
    def load_devices_page(self):
        for r in self.devices_table.get_children():
            self.devices_table.delete(r)

        try:
            conn = mysql.connector.connect(**DB_CONFIG)
            ensure_shifts_table(conn)
            cur = conn.cursor(dictionary=True)

            q = f"""
                SELECT
                    dt.id AS id_type,
                    dt.name AS device_type,
                    COUNT(d.id) AS production,
                    COALESCE(s.shifts, 2) AS shifts
                FROM devices d
                JOIN device_types dt ON d.id_type = dt.id
                LEFT JOIN `{SHIFTS_TABLE}` s ON s.id_type = dt.id
                WHERE d.status = 'Production'
                {self.location_sql_filter("d")}
                GROUP BY dt.id, dt.name, s.shifts
                ORDER BY production DESC;
            """

            cur.execute(q)
            rows = cur.fetchall()

            for i, r in enumerate(rows):
                tag = "even" if i % 2 == 0 else "odd"
                iid = str(r["id_type"])
                self.devices_table.insert(
                    "", "end", iid=iid,
                    values=(r["device_type"], r["production"], r["shifts"]),
                    tags=(tag,)
                )

            cur.close()
            conn.close()

        except Exception as e:
            print("DB ERROR PAGE2:", e)

    def _on_double_click_devices(self, event):
        region = self.devices_table.identify("region", event.x, event.y)
        if region != "cell":
            return

        column = self.devices_table.identify_column(event.x)
        col_index = int(column.replace("#", "")) - 1
        if col_index != 2:
            return

        row_id = self.devices_table.identify_row(event.y)
        if not row_id:
            return

        bbox = self.devices_table.bbox(row_id, column)
        if not bbox:
            return

        x, y, w, h = bbox
        current = self.devices_table.set(row_id, "shifts")

        if self._shift_editor:
            self._shift_editor.destroy()
            self._shift_editor = None

        cb = ttk.Combobox(self.devices_table, values=["1", "2", "3"], state="readonly")
        cb.place(x=x, y=y, width=w, height=h)
        cb.set(str(current))
        cb.focus()

        def save_and_close(_=None):
            val = cb.get()
            try:
                shifts = int(val)
                if shifts not in (1, 2, 3):
                    raise ValueError
            except Exception:
                messagebox.showerror("Eroare", "Ture trebuie să fie 1, 2 sau 3.")
                cb.destroy()
                return

            self.save_shifts_to_db(int(row_id), shifts)
            self.devices_table.set(row_id, "shifts", str(shifts))
            cb.destroy()
            self._shift_editor = None
            if self.page3.winfo_ismapped():
                self.load_year_hours_page()

        cb.bind("<<ComboboxSelected>>", save_and_close)
        cb.bind("<Return>", save_and_close)
        cb.bind("<FocusOut>", save_and_close)
        self._shift_editor = cb

    def save_shifts_to_db(self, id_type: int, shifts: int):
        try:
            conn = mysql.connector.connect(**DB_CONFIG)
            ensure_shifts_table(conn)
            cur = conn.cursor()
            cur.execute(
                f"""
                INSERT INTO `{SHIFTS_TABLE}` (id_type, shifts)
                VALUES (%s, %s)
                ON DUPLICATE KEY UPDATE shifts=VALUES(shifts), updated_at=NOW()
                """,
                (id_type, shifts)
            )
            conn.commit()
            cur.close()
            conn.close()
        except Exception as e:
            print("DB ERROR SAVE SHIFTS:", e)

    # ================= PAGE 3 =================
    def load_year_hours_page(self):
        yr = int(self.year_var.get())

        for r in self.year_table.get_children():
            self.year_table.delete(r)
        for w in self.leave_frame.winfo_children():
            w.destroy()

        try:
            conn = mysql.connector.connect(**DB_CONFIG)
            ensure_shifts_table(conn)
            holidays = get_holidays_set(conn, yr)

            month_rows = []
            total_workdays = 0
            for m in range(1, 13):
                wd = workdays_in_month(yr, m, holidays)
                total_workdays += wd
                t1 = wd * SHIFT_HOURS[1]
                t2 = wd * SHIFT_HOURS[2]
                t12 = wd * (SHIFT_HOURS[1] + SHIFT_HOURS[2])
                t123 = wd * (SHIFT_HOURS[1] + SHIFT_HOURS[2] + SHIFT_HOURS[3])
                month_rows.append((calendar.month_name[m], wd, t1, t2, t12, t123))

            lf = tk.Frame(self.leave_frame, bg="#111")
            lf.pack(fill="x")

            headers = ["Luna", "Zile lucrătoare", "Turul I", "Turul II", "Turul I+II", "Turul I+II+III"]
            for j, htxt in enumerate(headers):
                tk.Label(lf, text=htxt, bg="#333", fg="white",
                         font=("Arial", 12, "bold"), padx=8, pady=4).grid(row=0, column=j, sticky="nsew")

            for i, (mn, wd, t1, t2, t12, t123) in enumerate(month_rows, start=1):
                vals = [mn, wd, t1, t2, t12, t123]
                for j, v in enumerate(vals):
                    bg = "#f2f2f2" if i % 2 == 0 else "#ffffff"
                    tk.Label(lf, text=str(v), bg=bg, fg="#000",
                             font=("Arial", 12), padx=8, pady=3).grid(row=i, column=j, sticky="nsew")

            total_t1 = sum(r[2] for r in month_rows)
            total_t2 = sum(r[3] for r in month_rows)
            total_t12 = sum(r[4] for r in month_rows)
            total_t123 = sum(r[5] for r in month_rows)
            total_vals = ["Total", total_workdays, total_t1, total_t2, total_t12, total_t123]
            i = len(month_rows) + 1
            for j, v in enumerate(total_vals):
                tk.Label(lf, text=str(v), bg="#ddd", fg="#000",
                         font=("Arial", 12, "bold"), padx=8, pady=4).grid(row=i, column=j, sticky="nsew")

            cur = conn.cursor(dictionary=True)
            q = f"""
                SELECT
                    dt.id AS id_type,
                    dt.name AS device_type,
                    COUNT(d.id) AS production,
                    COALESCE(s.shifts, 2) AS shifts
                FROM devices d
                JOIN device_types dt ON d.id_type = dt.id
                LEFT JOIN `{SHIFTS_TABLE}` s ON s.id_type = dt.id
                WHERE d.status='Production'
                GROUP BY dt.id, dt.name, s.shifts
                ORDER BY dt.name ASC;
            """
            cur.execute(q)
            types = cur.fetchall()

            for idx, t in enumerate(types):
                prod = int(t.get("production") or 0)
                shifts = int(t.get("shifts") or 2)
                hpd = hours_per_day_by_shifts(shifts)

                month_hours = []
                for m in range(1, 13):
                    wd = month_rows[m - 1][1]
                    month_hours.append(wd * hpd * prod)

                total_year = sum(month_hours)
                tag = "even" if idx % 2 == 0 else "odd"

                self.year_table.insert(
                    "", "end",
                    values=(t["device_type"], prod, shifts, *month_hours, total_year),
                    tags=(tag,)
                )

            cur.close()
            conn.close()

        except Exception as e:
            print("DB ERROR PAGE3:", e)


# ================= RUN =================
if __name__ == "__main__":
    root = tk.Tk()
    app = GembaBoardApp(root)
    root.mainloop()
