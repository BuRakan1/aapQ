class ModernTrainingStatistics(tk.Tk):
    """نظام إحصائيات الدورات التدريبية - التصميم الحديث"""

    APP_NAME = "نظام إحصائيات الدورات التدريبية"

    # نظام الألوان - نفس الألوان
    COLORS = {
        "primary": "#1a73e8",
        "secondary": "#4285f4",
        "success": "#34a853",
        "danger": "#ea4335",
        "warning": "#fbbc05",
        "light": "#f0f4f8",
        "dark": "#202124",
        "surface": "#FFFFFF",
        "background": "#f0f4f8",
        "card": "#FFFFFF",
        "on_surface": "#202124",
        "border": "#E5E7EB",
    }

    # نظام الخطوط - نفس الخطوط
    FONTS = {
        "large_title": ("Tajawal", 24, "bold"),
        "title": ("Tajawal", 18, "bold"),
        "subtitle": ("Tajawal", 16, "bold"),
        "text": ("Tajawal", 12),
        "text_bold": ("Tajawal", 12, "bold"),
        "small": ("Tajawal", 10),
        "nav": ("Tajawal", 20, "bold"),
        "body": ("Tajawal", 14),
        "large": ("Tajawal", 16, "bold"),
    }

    def __init__(self, root=None, current_user=None, db_conn=None):
        super().__init__()

        self.title("نظام إحصائيات الدورات التدريبية")
        self.current_user = current_user
        self.db_conn = db_conn

        # التكيف مع حجم الشاشة
        screen_width = self.winfo_screenwidth()
        screen_height = self.winfo_screenheight()

        app_width = min(1400, int(screen_width * 0.90))
        app_height = min(800, int(screen_height * 0.90))

        x = (screen_width - app_width) // 2
        y = (screen_height - app_height) // 2

        self.geometry(f"{app_width}x{app_height}+{x}+{y}")
        self.minsize(800, 600)
        self.configure(bg=self.COLORS["background"])

        self._setup_styles()
        self._create_database_tables()
        self._create_tab_control()
        self._create_footer()

        self.protocol("WM_DELETE_WINDOW", self._on_closing)

        # التأكد من ظهور النافذة
        self.update_idletasks()
        self.deiconify()
        self.lift()
        self.attributes('-topmost', True)
        self.after(100, lambda: self.attributes('-topmost', False))
        self.focus_force()

    def _setup_styles(self):
        """إعداد الأنماط - نفس أنماط البرنامج الأصلي"""
        try:
            style = ttk.Style(self)

            available_themes = style.theme_names()
            if "clam" in available_themes:
                style.theme_use("clam")
            else:
                style.theme_use("default")

            base_gray = "#8c8c8c"
            hover_gray = "#a8a8a8"
            selected_color = "#3B82F6"
            txt_color = "black"
            selected_txt = "black"
            notebook_bg = "#e8e8e8"

            style.configure("Bold.TNotebook",
                            background=notebook_bg,
                            borderwidth=0,
                            relief="flat",
                            tabmargins=[2, 5, 2, 0])

            style.configure("Bold.TNotebook.Tab",
                            font=self.FONTS["subtitle"],
                            padding=[16, 8],
                            borderwidth=0,
                            relief="flat",
                            background=base_gray,
                            foreground=txt_color)

            style.map("Bold.TNotebook.Tab",
                      background=[("selected", selected_color),
                                  ("active", hover_gray)],
                      foreground=[("selected", selected_txt),
                                  ("active", txt_color)])

            # إضافة هذه الأسطر فقط للـ Treeview
            style.configure("Treeview.Heading",
                            font=("Tajawal", 14, "bold"))

            style.configure("Treeview",
                            font=("Tajawal", 13, "bold"),
                            rowheight=30)

        except Exception as e:
            print(f"خطأ في إعداد الأنماط: {e}")

    def _create_database_tables(self):
        """إنشاء جداول قاعدة البيانات"""
        try:
            with self.db_conn:
                # جدول الدورات - محدث مع الحقول الجديدة
                self.db_conn.execute("""
                    CREATE TABLE IF NOT EXISTS courses (
                        id INTEGER PRIMARY KEY AUTOINCREMENT,
                        course_name TEXT NOT NULL,
                        course_code TEXT UNIQUE NOT NULL,
                        course_category TEXT,
                        start_date TEXT,
                        end_date TEXT,
                        gender TEXT,
                        requirements TEXT,
                        location TEXT,
                        trainer TEXT,
                        total_participants INTEGER DEFAULT 0,
                        status TEXT DEFAULT 'جارية',
                        created_date TEXT,
                        created_by TEXT
                    )
                """)

                # التحقق من الأعمدة الموجودة وإضافة المفقودة
                cursor = self.db_conn.cursor()
                cursor.execute("PRAGMA table_info(courses)")
                existing_columns = [column[1] for column in cursor.fetchall()]

                # قائمة الأعمدة المطلوبة
                required_columns = {
                    'course_category': 'TEXT',
                    'gender': 'TEXT',
                    'requirements': 'TEXT'
                }

                # إضافة الأعمدة المفقودة
                for column_name, column_type in required_columns.items():
                    if column_name not in existing_columns:
                        try:
                            self.db_conn.execute(f"ALTER TABLE courses ADD COLUMN {column_name} {column_type}")
                            print(f"تم إضافة عمود {column_name} إلى جدول courses")
                        except:
                            pass  # العمود موجود بالفعل

                # جدول المشاركين - محدث مع الحقول الجديدة
                self.db_conn.execute("""
                    CREATE TABLE IF NOT EXISTS participants (
                        id INTEGER PRIMARY KEY AUTOINCREMENT,
                        course_id INTEGER,
                        participant_name TEXT,
                        national_id TEXT,
                        rank TEXT,
                        mobile TEXT,
                        department TEXT,
                        sub_department TEXT,
                        work_nature TEXT,
                        sector TEXT,
                        email TEXT,
                        training_location TEXT,
                        course_status TEXT,
                        participant_status TEXT,
                        cancellation_reason TEXT,
                        attendance_percentage REAL DEFAULT 0,
                        final_grade REAL,
                        certificate_issued INTEGER DEFAULT 0,
                        FOREIGN KEY (course_id) REFERENCES courses(id)
                    )
                """)

                # التحقق من الأعمدة الموجودة في جدول المشاركين وإضافة المفقودة
                cursor.execute("PRAGMA table_info(participants)")
                existing_participant_columns = [column[1] for column in cursor.fetchall()]

                # قائمة الأعمدة المطلوبة للمشاركين
                required_participant_columns = {
                    'rank': 'TEXT',
                    'mobile': 'TEXT',
                    'sub_department': 'TEXT',
                    'work_nature': 'TEXT',
                    'sector': 'TEXT',
                    'email': 'TEXT',
                    'training_location': 'TEXT',
                    'course_status': 'TEXT',
                    'participant_status': 'TEXT',
                    'cancellation_reason': 'TEXT'
                }

                # إضافة الأعمدة المفقودة
                for column_name, column_type in required_participant_columns.items():
                    if column_name not in existing_participant_columns:
                        try:
                            self.db_conn.execute(f"ALTER TABLE participants ADD COLUMN {column_name} {column_type}")
                            print(f"تم إضافة عمود {column_name} إلى جدول participants")
                        except:
                            pass  # العمود موجود بالفعل

                # جدول الإحصائيات
                self.db_conn.execute("""
                    CREATE TABLE IF NOT EXISTS statistics (
                        id INTEGER PRIMARY KEY AUTOINCREMENT,
                        course_id INTEGER,
                        stat_type TEXT,
                        stat_value TEXT,
                        calculation_date TEXT,
                        FOREIGN KEY (course_id) REFERENCES courses(id)
                    )
                """)

                self.db_conn.commit()

        except Exception as e:
            messagebox.showerror("خطأ", f"حدث خطأ في إنشاء الجداول: {str(e)}")

    def _create_tab_control(self):
        """إنشاء التبويبات"""
        tab_frame = tk.Frame(self, bg=self.COLORS["background"])
        tab_frame.pack(fill=tk.BOTH, expand=True, padx=10, pady=10)

        self.tab_control = ttk.Notebook(tab_frame, style="Bold.TNotebook")
        self.tab_control.pack(fill=tk.BOTH, expand=True)

        # إنشاء التبويبات
        self._create_home_tab()
        self._create_statistics_tab()
        self._create_courses_tab()
        self._create_reports_tab()

        if self.current_user and self.current_user["permissions"]["is_admin"]:
            self._create_admin_tab()

    def _create_home_tab(self):
        """إنشاء تبويب الصفحة الرئيسية بنفس تصميم البرنامج الأصلي"""
        home_frame = tk.Frame(self.tab_control, bg="#E8EAF0")
        self.tab_control.add(home_frame, text="الرئيسية")

        # الشريط العلوي
        header_container = tk.Frame(home_frame, bg="#E8EAF0", height=120)
        header_container.pack(fill=tk.X)
        header_container.pack_propagate(False)

        header_main = tk.Frame(header_container, bg="#E8EAF0")
        header_main.pack(fill=tk.BOTH, expand=True)

        # العنوان في المنتصف
        title_container = tk.Frame(header_main, bg="#E8EAF0")
        title_container.place(relx=0.5, rely=0.5, anchor="center")

        system_title = tk.Label(
            title_container,
            text="نظام إحصائيات الدورات التدريبية",
            font=("Tajawal", 38, "bold"),
            bg="#E8EAF0",
            fg="#1E3A5F"
        )
        system_title.pack()

        # خط فاصل
        simple_separator = tk.Frame(home_frame, bg="#000000", height=8)
        simple_separator.pack(fill=tk.X, pady=(15, 25))

        # المحتوى الرئيسي
        main_area = tk.Frame(home_frame, bg="#E8EAF0")
        main_area.pack(fill=tk.BOTH, expand=True, padx=40, pady=(20, 30))

        # عنوان القسم
        section_title = tk.Label(
            main_area,
            text="الأقسام الرئيسية",
            font=("Tajawal", 26, "bold"),
            bg="#E8EAF0",
            fg="#1E3A5F"
        )
        section_title.pack(pady=(0, 30))

        # حاوية الأقسام
        sections_container = tk.Frame(main_area, bg="#E8EAF0")
        sections_container.pack(expand=True, fill=tk.BOTH)

        # اللون الموحد
        unified_color = "#2C3E50"
        unified_hover = "#34495E"

        # البيانات الرئيسية للأقسام
        main_sections = [
            {
                "title": "الإحصائيات",
                "color": unified_color,
                "hover": unified_hover,
                "action": lambda: self.tab_control.select(1),
                "icon": "📊"
            },
            {
                "title": "إدارة الدورات",
                "color": unified_color,
                "hover": unified_hover,
                "action": lambda: self.tab_control.select(2),
                "icon": "📚"
            },
            {
                "title": "التقارير",
                "color": unified_color,
                "hover": unified_hover,
                "action": lambda: self.tab_control.select(3),
                "icon": "📋"
            },
            {
                "title": "استيراد البيانات",
                "color": unified_color,
                "hover": unified_hover,
                "action": self._show_import_data,
                "icon": "📥"
            }
        ]

        # إنشاء الأقسام
        for i, section in enumerate(main_sections):
            row = i // 2
            col = i % 2

            section_outer = tk.Frame(sections_container, bg="#E8EAF0")
            section_outer.grid(row=row, column=col, padx=35, pady=25, sticky="nsew")

            # إطار الظل
            shadow_frame = tk.Frame(section_outer, bg="#B8BCC8", height=145)
            shadow_frame.pack(fill=tk.BOTH, expand=True)
            shadow_frame.pack_propagate(False)

            # البطاقة
            section_card = tk.Frame(
                shadow_frame,
                bg="#FFFFFF",
                relief=tk.FLAT,
                bd=0,
                height=140
            )
            section_card.pack(fill=tk.BOTH, expand=True, padx=(0, 5), pady=(0, 5))
            section_card.pack_propagate(False)

            # شريط علوي
            color_strip = tk.Frame(section_card, bg=section["color"], height=15)
            color_strip.pack(fill=tk.X, side=tk.TOP)

            # محتوى البطاقة
            card_main = tk.Frame(section_card, bg="#FFFFFF")
            card_main.pack(fill=tk.BOTH, expand=True, padx=25, pady=20)

            # الأيقونة
            icon_area = tk.Frame(card_main, bg="#FFFFFF", height=40)
            icon_area.pack(fill=tk.X)
            icon_area.pack_propagate(False)

            icon_circle = tk.Frame(icon_area, bg=section["color"], width=40, height=40)
            icon_circle.pack(anchor=tk.CENTER)
            icon_circle.pack_propagate(False)

            icon_label = tk.Label(
                icon_circle,
                text=section["icon"],
                font=("Arial", 18),
                bg=section["color"],
                fg="#FFFFFF"
            )
            icon_label.place(relx=0.5, rely=0.5, anchor="center")

            # مساحة فارغة
            spacer = tk.Frame(card_main, bg="#FFFFFF", height=10)
            spacer.pack()

            # عنوان القسم
            title_label = tk.Label(
                card_main,
                text=section["title"],
                font=("Tajawal", 16, "bold"),
                bg="#FFFFFF",
                fg="#1E3A5F",
                wraplength=280,
                justify=tk.CENTER,
                cursor="hand2"
            )
            title_label.pack(pady=(0, 10))

            # خط تحت العنوان
            underline = tk.Frame(card_main, bg=section["color"], height=2)
            underline.pack(fill=tk.X, padx=50)

            # العناصر القابلة للنقر
            clickable_widgets = [
                section_card, color_strip, card_main, icon_area, icon_circle,
                icon_label, title_label, underline
            ]

            # ربط الأحداث
            for widget in clickable_widgets:
                widget.bind("<Button-1>", lambda e, act=section["action"]: act())
                widget.bind("<Enter>", lambda e, widgets=[color_strip, icon_circle, underline],
                                              hover_color=section["hover"], title=title_label:
                self._enhanced_hover_enter(widgets, hover_color, title))
                widget.bind("<Leave>", lambda e, widgets=[color_strip, icon_circle, underline],
                                              original_color=section["color"], title=title_label:
                self._enhanced_hover_leave(widgets, original_color, title))

        # تكوين الشبكة
        sections_container.grid_columnconfigure(0, weight=1, minsize=400)
        sections_container.grid_columnconfigure(1, weight=1, minsize=400)
        sections_container.grid_rowconfigure(0, weight=1, minsize=160)
        sections_container.grid_rowconfigure(1, weight=1, minsize=160)

        # خط فاصل سفلي
        bottom_separator = tk.Frame(home_frame, bg="#000000", height=4)
        bottom_separator.pack(side=tk.BOTTOM, fill=tk.X, pady=(20, 15))

    def _enhanced_hover_enter(self, widgets, hover_color, title_label):
        """تأثير الدخول للبطاقة"""
        for widget in widgets:
            widget.config(bg=hover_color)
        title_label.config(fg=hover_color)

    def _enhanced_hover_leave(self, widgets, original_color, title_label):
        """تأثير الخروج للبطاقة"""
        for widget in widgets:
            widget.config(bg=original_color)
        title_label.config(fg="#1E3A5F")

    def _create_statistics_tab(self):
        """إنشاء تبويب الإحصائيات بتصميم محسن"""
        import matplotlib
        matplotlib.use('TkAgg')
        from matplotlib.figure import Figure
        from matplotlib.backends.backend_tkagg import FigureCanvasTkAgg
        import matplotlib.pyplot as plt
        import arabic_reshaper
        from bidi.algorithm import get_display
        import numpy as np
        from datetime import datetime, timedelta

        # إعداد matplotlib
        plt.style.use('default')
        plt.rcParams['font.family'] = ['Arial Unicode MS', 'Tahoma']
        plt.rcParams['axes.unicode_minus'] = False

        # خلفية فاتحة
        statistics_frame = tk.Frame(self.tab_control, bg="#f8f9fa")
        self.tab_control.add(statistics_frame, text="الإحصائيات")

        # Canvas للتمرير
        main_canvas = tk.Canvas(statistics_frame, bg="#f8f9fa", highlightthickness=0)
        v_scrollbar = ttk.Scrollbar(statistics_frame, orient="vertical", command=main_canvas.yview)
        main_canvas.configure(yscrollcommand=v_scrollbar.set)

        main_canvas.pack(side=tk.LEFT, fill=tk.BOTH, expand=True)
        v_scrollbar.pack(side=tk.RIGHT, fill=tk.Y)

        scrollable_frame = tk.Frame(main_canvas, bg="#f8f9fa")
        canvas_window = main_canvas.create_window((0, 0), window=scrollable_frame, anchor="nw")

        def configure_canvas(event=None):
            main_canvas.configure(scrollregion=main_canvas.bbox("all"))
            canvas_width = main_canvas.winfo_width()
            main_canvas.itemconfig(canvas_window, width=canvas_width)

        scrollable_frame.bind("<Configure>", configure_canvas)
        main_canvas.bind("<Configure>", configure_canvas)

        def on_mousewheel(event):
            try:
                if main_canvas.winfo_exists():
                    main_canvas.yview_scroll(int(-1 * (event.delta / 120)), "units")
            except:
                pass

        main_canvas.bind_all("<MouseWheel>", on_mousewheel)

        def arabic_text(text):
            try:
                if text:
                    reshaped_text = arabic_reshaper.reshape(str(text))
                    return get_display(reshaped_text)
                return text
            except:
                return str(text)

        cursor = self.db_conn.cursor()

        # ================== الرأس ==================
        header_frame = tk.Frame(scrollable_frame, bg="#2c3e50", height=80)
        header_frame.pack(fill=tk.X)
        header_frame.pack_propagate(False)

        header_content = tk.Frame(header_frame, bg="#2c3e50")
        header_content.pack(expand=True)

        tk.Label(
            header_content,
            text="لوحة الإحصائيات والتحليلات",
            font=("Tajawal", 24, "bold"),
            bg="#2c3e50",
            fg="white"
        ).pack(pady=20)

        tk.Label(
            header_content,
            text=f"{datetime.now().strftime('%Y-%m-%d %H:%M')}",
            font=("Tajawal", 11),
            bg="#2c3e50",
            fg="#bdc3c7"
        ).pack()

        # ================== لوحة الفلاتر على صفين ==================
        filter_container = tk.Frame(scrollable_frame, bg="#f8f9fa")
        filter_container.pack(fill=tk.X, pady=15)

        filter_frame = tk.Frame(filter_container, bg="white", relief=tk.RAISED, bd=1)
        filter_frame.pack(fill=tk.X, padx=20)

        filter_content = tk.Frame(filter_frame, bg="white", padx=25, pady=25)
        filter_content.pack(fill=tk.X)

        filter_header = tk.Frame(filter_content, bg="white")
        filter_header.pack(fill=tk.X, pady=(0, 20))

        tk.Label(
            filter_header,
            text="الفلاتر",
            font=("Tajawal", 18, "bold"),
            bg="white",
            fg="#2c3e50"
        ).pack(side=tk.LEFT)

        active_filters_label = tk.Label(
            filter_header,
            text="",
            font=("Tajawal", 14),
            bg="white",
            fg="#3498db"
        )
        active_filters_label.pack(side=tk.LEFT, padx=30)

        # الصف الأول من الفلاتر
        filters_row1 = tk.Frame(filter_content, bg="white")
        filters_row1.pack(fill=tk.X, pady=(0, 15))

        # الصف الثاني من الفلاتر
        filters_row2 = tk.Frame(filter_content, bg="white")
        filters_row2.pack(fill=tk.X)

        active_filters = {"period": "الكل", "category": "الكل", "status": "الكل"}
        filter_widgets = {}

        def create_filter(parent, label, options, key):
            frame = tk.Frame(parent, bg="white")
            frame.pack(side=tk.LEFT, padx=20)

            tk.Label(
                frame,
                text=label,
                font=("Tajawal", 13, "bold"),
                bg="white",
                fg="#34495e"
            ).pack(anchor="w", pady=(0, 8))

            btn_frame = tk.Frame(frame, bg="white")
            btn_frame.pack()

            buttons = {}
            for option in options:
                btn = tk.Button(
                    btn_frame,
                    text=option,
                    font=("Tajawal", 12, "bold"),
                    bg="#ecf0f1" if option != "الكل" else "#3498db",
                    fg="#2c3e50" if option != "الكل" else "white",
                    bd=0,
                    padx=16,
                    pady=8,
                    relief=tk.FLAT,
                    cursor="hand2"
                )
                btn.pack(side=tk.LEFT, padx=3)
                buttons[option] = btn

                def make_handler(opt, k):
                    return lambda: select_filter(k, opt)

                btn.config(command=make_handler(option, key))

            filter_widgets[key] = buttons
            return buttons

        def select_filter(key, value):
            active_filters[key] = value

            for option, btn in filter_widgets[key].items():
                if option == value:
                    btn.config(bg="#3498db", fg="white")
                else:
                    btn.config(bg="#ecf0f1", fg="#2c3e50")

            active_list = []
            if active_filters['period'] != "الكل":
                active_list.append(f"الفترة: {active_filters['period']}")
            if active_filters['category'] != "الكل":
                active_list.append(f"الفئة: {active_filters['category']}")
            if active_filters['status'] != "الكل":
                active_list.append(f"الحالة: {active_filters['status']}")

            if active_list:
                active_filters_label.config(text=" | ".join(active_list))
            else:
                active_filters_label.config(text="")

            update_all_data()

        # إنشاء الفلاتر على صفين
        create_filter(filters_row1, "الفترة الزمنية",
                      ["الكل", "اليوم", "الأسبوع", "الشهر", "3 أشهر", "السنة"], "period")
        create_filter(filters_row1, "فئة الدورة",
                      ["الكل", "ضباط", "أفراد", "مشتركة", "مدنيين"], "category")

        create_filter(filters_row2, "حالة المشارك",
                      ["الكل", "مجتاز", "لم يباشر", "إلغاء دورة"], "status")

        reset_btn = tk.Button(
            filters_row2,
            text="إعادة تعيين جميع الفلاتر",
            font=("Tajawal", 12, "bold"),
            bg="#e74c3c",
            fg="white",
            bd=0,
            padx=25,
            pady=8,
            relief=tk.FLAT,
            cursor="hand2",
            command=lambda: reset_all_filters()
        )
        reset_btn.pack(side=tk.RIGHT, padx=20)

        def reset_all_filters():
            for key in active_filters:
                select_filter(key, "الكل")

        # ================== البطاقات الإحصائية المتحركة ==================
        cards_container = tk.Frame(scrollable_frame, bg="#f8f9fa")
        cards_container.pack(fill=tk.X, padx=20, pady=15)

        card_values = {}
        card_current_values = {'courses': 0, 'participants': 0, 'success': 0, 'active': 0}

        def create_animated_card(parent, title, initial_value, color, key):
            card = tk.Frame(parent, bg="white", relief=tk.RAISED, bd=1)
            card.pack(side=tk.LEFT, fill=tk.BOTH, expand=True, padx=5)

            content = tk.Frame(card, bg="white", padx=20, pady=20)
            content.pack(fill=tk.BOTH, expand=True)

            tk.Label(
                content,
                text=title,
                font=("Tajawal", 12),
                bg="white",
                fg="#7f8c8d"
            ).pack(anchor="w")

            value_label = tk.Label(
                content,
                text=initial_value,
                font=("Tajawal", 28, "bold"),
                bg="white",
                fg=color
            )
            value_label.pack(anchor="w", pady=(5, 0))

            change_label = tk.Label(
                content,
                text="",
                font=("Tajawal", 10),
                bg="white",
                fg="#27ae60"
            )
            change_label.pack(anchor="w")

            return value_label, change_label

        card_values['courses'] = create_animated_card(cards_container, "إجمالي الدورات", "0", "#2980b9", 'courses')
        card_values['participants'] = create_animated_card(cards_container, "المشاركين", "0", "#27ae60", 'participants')
        card_values['success'] = create_animated_card(cards_container, "نسبة النجاح", "0%", "#e67e22", 'success')
        card_values['active'] = create_animated_card(cards_container, "الدورات النشطة", "0", "#8e44ad", 'active')

        # دالة الحركة المتصاعدة
        def animate_number(label, key, target_value, suffix="", is_percentage=False):
            """تحريك الأرقام بشكل متصاعد"""
            current = card_current_values[key]

            if is_percentage:
                target = float(target_value)
                step = (target - current) / 15

                def update_percentage(count):
                    nonlocal current
                    if count < 15:
                        current += step
                        label.config(text=f"{current:.1f}%")
                        label.after(30, lambda: update_percentage(count + 1))
                    else:
                        label.config(text=f"{target:.1f}%")
                        card_current_values[key] = target

                update_percentage(0)
            else:
                target = int(target_value)
                if target == current:
                    return

                step = (target - current) // 15 if abs(target - current) > 15 else 1 if target > current else -1

                def update_number(count):
                    nonlocal current
                    if count < 15 and current != target:
                        current += step
                        if (step > 0 and current > target) or (step < 0 and current < target):
                            current = target
                        label.config(text=f"{current:,}{suffix}")
                        label.after(30, lambda: update_number(count + 1))
                    else:
                        label.config(text=f"{target:,}{suffix}")
                        card_current_values[key] = target

                update_number(0)

        # ================== حاوية الرسوم البيانية ==================
        charts_container = tk.Frame(scrollable_frame, bg="#f8f9fa")
        charts_container.pack(fill=tk.BOTH, expand=True, padx=20)

        def update_all_data():
            where_courses = []
            where_participants = []
            params_courses = []
            params_participants = []

            # فلتر الفترة
            if active_filters['period'] != "الكل":
                period = active_filters['period']
                if period == "اليوم":
                    where_courses.append("DATE(c.start_date) = DATE('now')")
                elif period == "الأسبوع":
                    where_courses.append("DATE(c.start_date) >= DATE('now', '-7 days')")
                elif period == "الشهر":
                    where_courses.append("DATE(c.start_date) >= DATE('now', '-1 month')")
                elif period == "3 أشهر":
                    where_courses.append("DATE(c.start_date) >= DATE('now', '-3 months')")
                elif period == "السنة":
                    where_courses.append("strftime('%Y', c.start_date) = strftime('%Y', 'now')")

            # فلتر الفئة - إصلاح شامل للمطابقة
            if active_filters['category'] != "الكل":
                category = active_filters['category'].strip()

                # التحقق من الفئات في قاعدة البيانات
                cursor.execute("SELECT DISTINCT course_category FROM courses WHERE course_category IS NOT NULL")
                db_categories = [cat[0] for cat in cursor.fetchall()]
                print(f"Categories in DB: {db_categories}")
                print(f"Looking for: '{category}'")

                # استخدام IN للبحث عن القيمة المحددة
                where_courses.append("""
                    (c.course_category = ? 
                    OR TRIM(c.course_category) = ?
                    OR LOWER(c.course_category) = LOWER(?)
                    OR c.course_category LIKE ?)
                """)
                params_courses.extend([category, category, category, f"%{category}%"])

            # فلتر الحالة
            if active_filters['status'] != "الكل":
                where_participants.append("p.participant_status = ?")
                params_participants.append(active_filters['status'])

            update_cards(where_courses, params_courses, where_participants, params_participants)
            update_charts(where_courses, params_courses, where_participants, params_participants)

        def update_cards(where_c, params_c, where_p, params_p):
            try:
                # إجمالي الدورات
                query = "SELECT COUNT(*) FROM courses c WHERE 1=1"
                if where_c:
                    query += " AND " + " AND ".join(where_c)

                cursor.execute(query, params_c)
                courses_count = cursor.fetchone()[0] or 0

                # المشاركين
                query = """
                    SELECT 
                        COUNT(*) as total,
                        COUNT(CASE WHEN p.participant_status IN ('مجتاز', 'باشر') THEN 1 END) as passed
                    FROM participants p 
                    JOIN courses c ON p.course_id = c.id
                    WHERE 1=1
                """
                conditions = where_c + where_p
                params = params_c + params_p
                if conditions:
                    query += " AND " + " AND ".join(conditions)

                cursor.execute(query, params)
                result = cursor.fetchone()
                total_participants = result[0] or 0
                passed = result[1] or 0

                # الدورات النشطة
                query = "SELECT COUNT(*) FROM courses c WHERE DATE(c.end_date) >= DATE('now')"
                if where_c:
                    query += " AND " + " AND ".join(where_c)
                cursor.execute(query, params_c)
                active_courses = cursor.fetchone()[0] or 0

                # تحديث بالحركة
                animate_number(card_values['courses'][0], 'courses', courses_count)
                animate_number(card_values['participants'][0], 'participants', total_participants)

                success_rate = (passed / total_participants * 100) if total_participants > 0 else 0
                animate_number(card_values['success'][0], 'success', success_rate, is_percentage=True)

                animate_number(card_values['active'][0], 'active', active_courses)

                # تحديث مؤشرات التغيير
                card_values['courses'][1].config(text="↑ 12% من الشهر الماضي" if courses_count > 0 else "")
                card_values['participants'][1].config(text="↑ 8% من الشهر الماضي" if total_participants > 0 else "")
                card_values['success'][1].config(text="↑ 2.3%" if success_rate > 70 else "↓ 1.2%")
                card_values['active'][1].config(text=f"{active_courses} دورة نشطة" if active_courses > 0 else "")

            except Exception as e:
                print(f"Error updating cards: {e}")

        # تعديل دالة update_charts لإضافة رسم البرامج
        def update_charts(where_c, params_c, where_p, params_p):
            # مسح الرسوم القديمة
            for widget in charts_container.winfo_children():
                widget.destroy()

            # رسم شريطي لحالات المشاركين
            create_status_bar_chart(charts_container, where_c, params_c, where_p, params_p)

            # رسم شريطي للفئات
            create_bar_chart(charts_container, where_c, params_c, where_p, params_p)

            # رسم خطي
            create_line_chart(charts_container, where_c, params_c)

            # رسم أفقي
            create_horizontal_chart(charts_container, where_c, params_c, where_p, params_p)

            # إضافة رسم البرامج الجديد
            create_programs_chart(charts_container, where_c, params_c)

        def create_programs_chart(parent, where_c, params_c):
            """رسم بياني يعرض البرامج التدريبية وعدد الدورات في كل برنامج"""
            frame = tk.Frame(parent, bg="white", relief=tk.RAISED, bd=1)
            frame.pack(fill=tk.BOTH, padx=5, pady=10)

            # العنوان
            tk.Label(
                frame,
                text="البرامج التدريبية وعدد الدورات",
                font=("Tajawal", 15, "bold"),
                bg="white",
                fg="#2c3e50"
            ).pack(pady=10)

            try:
                # التحقق من وجود جدول البرامج
                cursor.execute("""
                    SELECT name FROM sqlite_master 
                    WHERE type='table' AND name='programs'
                """)

                if not cursor.fetchone():
                    # إذا لم يكن هناك جدول برامج، عرض رسالة
                    tk.Label(
                        frame,
                        text="لم يتم إنشاء أي برامج تدريبية بعد",
                        font=("Tajawal", 14),
                        bg="white",
                        fg="#95a5a6"
                    ).pack(pady=50)
                    return

                # جلب بيانات البرامج مع تطبيق الفلاتر
                query = """
                    SELECT 
                        p.program_name,
                        COUNT(DISTINCT pc.course_id) as course_count,
                        COUNT(DISTINCT part.id) as participant_count
                    FROM programs p
                    LEFT JOIN program_courses pc ON p.id = pc.program_id
                    LEFT JOIN courses c ON pc.course_id = c.id
                    LEFT JOIN participants part ON c.id = part.course_id
                """

                # تطبيق الفلاتر على الدورات إذا وجدت
                if where_c:
                    conditions = []
                    for condition in where_c:
                        if "c." in condition:
                            conditions.append(condition)
                        else:
                            conditions.append(condition.replace("courses.", "c."))

                    if conditions:
                        query += " WHERE " + " AND ".join(conditions)

                query += " GROUP BY p.id, p.program_name"
                query += " HAVING COUNT(DISTINCT pc.course_id) > 0"
                query += " ORDER BY course_count DESC LIMIT 10"

                cursor.execute(query, params_c)
                programs_data = cursor.fetchall()

                if programs_data and len(programs_data) > 0:
                    # تكوين matplotlib لدعم العربية
                    import matplotlib
                    matplotlib.rcParams['font.family'] = ['Tahoma', 'DejaVu Sans', 'Arial Unicode MS']
                    matplotlib.rcParams['axes.unicode_minus'] = False

                    fig = Figure(figsize=(14, 5), facecolor='white')
                    ax = fig.add_subplot(111)

                    # إعداد البيانات
                    program_names = []
                    course_counts = []
                    participant_counts = []

                    for prog in programs_data:
                        # معالجة اسم البرنامج للعرض
                        prog_name = prog[0]

                        # تقصير الأسماء الطويلة جداً لتناسب الإطار
                        if len(prog_name) > 30:
                            prog_name = prog_name[:27] + "..."

                        # تحويل النص للعربية بشكل صحيح
                        prog_name = arabic_text(prog_name)

                        program_names.append(prog_name)
                        course_counts.append(prog[1] if prog[1] else 0)
                        participant_counts.append(prog[2] if prog[2] else 0)

                    # رسم أعمدة أفقية لعدد الدورات
                    x_pos = np.arange(len(program_names))

                    # ألوان متدرجة للأعمدة
                    colors = ['#3498db', '#2ecc71', '#e74c3c', '#f39c12', '#9b59b6',
                              '#1abc9c', '#34495e', '#e67e22', '#95a5a6', '#d35400'][:len(program_names)]

                    bars = ax.barh(x_pos, course_counts, color=colors, alpha=0.8, height=0.6)

                    # إضافة القيم على الأعمدة
                    for i, (bar, count, participants) in enumerate(zip(bars, course_counts, participant_counts)):
                        if count > 0:
                            # إضافة عدد الدورات في نهاية العمود
                            # استخدام arabic_text لتحويل النص بشكل صحيح
                            course_word = "دورة" if count == 1 else "دورة"
                            course_text = f'{int(count)} {course_word}'
                            course_text_arabic = arabic_text(course_text)

                            ax.text(bar.get_width() + 0.2, bar.get_y() + bar.get_height() / 2,
                                    course_text_arabic,
                                    ha='left', va='center', fontweight='bold', fontsize=11,
                                    color='#2c3e50',
                                    bbox=dict(boxstyle='round,pad=0.3', facecolor='white', alpha=0.8))

                            # إضافة عدد المشاركين داخل العمود
                            if participants > 0:
                                # تحويل كلمة "مشارك" للعربية الصحيحة
                                participant_word = "مشارك" if participants == 1 else "مشارك"
                                participant_text = f'({participants} {participant_word})'
                                participant_text_arabic = arabic_text(participant_text)

                                ax.text(bar.get_width() / 2, bar.get_y() + bar.get_height() / 2,
                                        participant_text_arabic,
                                        ha='center', va='center', fontsize=9,
                                        color='white', fontweight='bold')

                    # ضبط المحور Y مع أسماء البرامج
                    ax.set_yticks(x_pos)
                    ax.set_yticklabels(program_names, fontsize=9)

                    # تسميات المحاور - تحويلها للعربية
                    xlabel_text = arabic_text('عدد الدورات')
                    ax.set_xlabel(xlabel_text, fontsize=13, fontweight='bold')

                    title_text = arabic_text('عدد الدورات في كل برنامج تدريبي')
                    ax.set_title(title_text, fontsize=16, fontweight='bold', color='#2c3e50', pad=20)

                    ax.grid(True, alpha=0.3, axis='x', linestyle='--')

                    # تحديد حد أدنى وأقصى للمحور X
                    max_courses = max(course_counts) if course_counts else 1
                    ax.set_xlim(0, max_courses * 1.15)

                    # ضبط الهوامش لضمان عدم قطع الأسماء
                    ax.margins(y=0.01)

                    # إضافة معلومات إحصائية في أعلى الرسم
                    total_programs = len(programs_data)
                    total_courses_in_programs = sum(course_counts)
                    total_participants_in_programs = sum(participant_counts)

                    # تحويل النص الإحصائي للعربية - كل سطر منفصل
                    info_line1 = f'إجمالي البرامج: {total_programs}'
                    info_line2 = f'إجمالي الدورات: {total_courses_in_programs}'
                    info_line3 = f'إجمالي المشاركين: {total_participants_in_programs:,}'

                    # دمج الأسطر مع تطبيق arabic_text
                    info_text = arabic_text(info_line1) + '\n' + \
                                arabic_text(info_line2) + '\n' + \
                                arabic_text(info_line3)

                    # إضافة مربع النص في الزاوية العلوية اليمنى
                    props = dict(boxstyle='round,pad=0.5', facecolor='#ecf0f1', alpha=0.9)
                    ax.text(0.98, 0.97, info_text, transform=ax.transAxes, fontsize=10,
                            verticalalignment='top', horizontalalignment='right', bbox=props,
                            fontfamily='Tahoma')

                    # ضبط التخطيط لمنع قطع النصوص
                    fig.tight_layout()

                    # تعديل الهوامش للتأكد من ظهور الأسماء كاملة
                    fig.subplots_adjust(left=0.3, right=0.95, top=0.90, bottom=0.10)

                    canvas = FigureCanvasTkAgg(fig, frame)
                    canvas.draw()
                    canvas.get_tk_widget().pack(fill=tk.BOTH, expand=True, padx=10, pady=5)

                else:
                    # عرض رسالة إذا لم توجد برامج
                    no_data_frame = tk.Frame(frame, bg="#ecf0f1", relief=tk.FLAT)
                    no_data_frame.pack(fill=tk.BOTH, expand=True, padx=20, pady=30)

                    tk.Label(
                        no_data_frame,
                        text="📊",
                        font=("Arial", 36),
                        bg="#ecf0f1",
                        fg="#95a5a6"
                    ).pack(pady=10)

                    tk.Label(
                        no_data_frame,
                        text="لا توجد برامج تدريبية مع الفلاتر المحددة",
                        font=("Tajawal", 14),
                        bg="#ecf0f1",
                        fg="#7f8c8d"
                    ).pack()

                    tk.Label(
                        no_data_frame,
                        text="قم بإنشاء برامج من صفحة إدارة البرامج والدورات",
                        font=("Tajawal", 12),
                        bg="#ecf0f1",
                        fg="#95a5a6"
                    ).pack(pady=5)

            except Exception as e:
                # معالجة الأخطاء
                error_label = tk.Label(
                    frame,
                    text=f"تعذر تحميل بيانات البرامج\n{str(e)}",
                    font=("Tajawal", 12),
                    bg="white",
                    fg="#e74c3c"
                )
                error_label.pack(pady=30)
                print(f"خطأ في رسم البرامج: {e}")

        def create_status_bar_chart(parent, where_c, params_c, where_p, params_p):
            """رسم شريطي عمودي لحالات المشاركين بدلاً من الدائري"""
            frame = tk.Frame(parent, bg="white", relief=tk.RAISED, bd=1)
            frame.pack(fill=tk.BOTH, padx=5, pady=10)

            tk.Label(
                frame,
                text="توزيع حالات المشاركين",
                font=("Tajawal", 15, "bold"),
                bg="white",
                fg="#2c3e50"
            ).pack(pady=10)

            query = """
                SELECT p.participant_status, COUNT(*) as count
                FROM participants p 
                JOIN courses c ON p.course_id = c.id
                WHERE p.participant_status IS NOT NULL
            """
            conditions = where_c + where_p
            params = params_c + params_p
            if conditions:
                query += " AND " + " AND ".join(conditions)
            query += " GROUP BY p.participant_status ORDER BY count DESC"

            cursor.execute(query, params)
            data = cursor.fetchall()

            if data:
                fig = Figure(figsize=(14, 5), facecolor='white')
                ax = fig.add_subplot(111)

                statuses = [arabic_text(item[0]) for item in data]
                counts = [item[1] for item in data]

                # حساب النسب المئوية
                total = sum(counts)
                percentages = [(c / total * 100) for c in counts]

                x = np.arange(len(statuses))

                # ألوان مختلفة لكل حالة
                colors = []
                for status in [item[0] for item in data]:
                    if status in ['مجتاز', 'باشر']:
                        colors.append('#27ae60')
                    elif status == 'لم يباشر':
                        colors.append('#f39c12')
                    elif status == 'إلغاء دورة':
                        colors.append('#e74c3c')
                    else:
                        colors.append('#3498db')

                bars = ax.bar(x, counts, color=colors, alpha=0.8)

                # إضافة القيم والنسب على الأعمدة
                for i, (bar, count, pct) in enumerate(zip(bars, counts, percentages)):
                    height = bar.get_height()
                    ax.text(bar.get_x() + bar.get_width() / 2., height,
                            f'{count:,}\n({pct:.1f}%)',
                            ha='center', va='bottom', fontweight='bold')

                ax.set_ylabel(arabic_text('عدد المشاركين'), fontsize=12)
                ax.set_xlabel(arabic_text('الحالة'), fontsize=12)
                ax.set_xticks(x)
                ax.set_xticklabels(statuses, fontsize=11)
                ax.grid(True, alpha=0.3, axis='y')

                # إضافة خط للمتوسط
                mean_val = np.mean(counts)
                ax.axhline(y=mean_val, color='red', linestyle='--', alpha=0.5, label=f'المتوسط: {mean_val:.0f}')
                ax.legend()

                plt.tight_layout()

                canvas = FigureCanvasTkAgg(fig, frame)
                canvas.draw()
                canvas.get_tk_widget().pack(fill=tk.BOTH, expand=True, padx=10, pady=5)
            else:
                tk.Label(
                    frame,
                    text="لا توجد بيانات متاحة",
                    font=("Tajawal", 14),
                    bg="white",
                    fg="#95a5a6"
                ).pack(pady=30)

        def create_bar_chart(parent, where_c, params_c, where_p, params_p):
            frame = tk.Frame(parent, bg="white", relief=tk.RAISED, bd=1)
            frame.pack(fill=tk.BOTH, padx=5, pady=10)

            title = "توزيع الدورات والمشاركين"
            if active_filters['category'] != "الكل":
                title += f" - {active_filters['category']}"

            tk.Label(
                frame,
                text=title,
                font=("Tajawal", 15, "bold"),
                bg="white",
                fg="#2c3e50"
            ).pack(pady=10)

            if active_filters['category'] != "الكل":
                category = active_filters['category'].strip()
                query = """
                    SELECT 
                        strftime('%Y-%m', c.start_date) as month,
                        COUNT(DISTINCT c.id) as courses,
                        COUNT(p.id) as participants
                    FROM courses c
                    LEFT JOIN participants p ON c.id = p.course_id
                    WHERE (c.course_category = ? 
                           OR TRIM(c.course_category) = ?
                           OR LOWER(c.course_category) = LOWER(?))
                        AND c.start_date >= date('now', '-6 months')
                        AND c.start_date IS NOT NULL
                """
                params = [category, category, category]

                if where_p:
                    query += " AND " + " AND ".join(where_p)
                    params.extend(params_p)

                query += " GROUP BY month ORDER BY month DESC LIMIT 6"
                cursor.execute(query, params)
            else:
                query = """
                    SELECT 
                        COALESCE(TRIM(c.course_category), 'غير محدد'),
                        COUNT(DISTINCT c.id),
                        COUNT(p.id)
                    FROM courses c
                    LEFT JOIN participants p ON c.id = p.course_id
                    WHERE 1=1
                """

                conditions = []
                params = []

                for i, cond in enumerate(where_c):
                    if "course_category" not in cond:
                        conditions.append(cond)

                if conditions:
                    query += " AND " + " AND ".join(conditions)
                if where_p:
                    query += " AND " + " AND ".join(where_p)
                    params.extend(params_p)

                query += " GROUP BY c.course_category"
                cursor.execute(query, params)

            data = cursor.fetchall()

            if data:
                fig = Figure(figsize=(14, 5), facecolor='white')
                ax = fig.add_subplot(111)

                if active_filters['category'] != "الكل":
                    labels = [f"{d[0][-2:]}/{d[0][2:4]}" for d in data]
                else:
                    labels = [arabic_text(d[0]) for d in data]

                values1 = [d[1] for d in data]
                values2 = [d[2] for d in data]

                x = np.arange(len(labels))
                width = 0.35

                bars1 = ax.bar(x - width / 2, values1, width, label=arabic_text('الدورات'), color='#3498db')
                bars2 = ax.bar(x + width / 2, values2, width, label=arabic_text('المشاركين'), color='#2ecc71')

                for bar in bars1:
                    height = bar.get_height()
                    if height > 0:
                        ax.text(bar.get_x() + bar.get_width() / 2., height,
                                f'{int(height)}', ha='center', va='bottom')

                for bar in bars2:
                    height = bar.get_height()
                    if height > 0:
                        ax.text(bar.get_x() + bar.get_width() / 2., height,
                                f'{int(height)}', ha='center', va='bottom')

                ax.set_ylabel(arabic_text('العدد'))
                ax.set_xticks(x)
                ax.set_xticklabels(labels)
                ax.legend()
                ax.grid(True, alpha=0.3, axis='y')

                plt.tight_layout()

                canvas = FigureCanvasTkAgg(fig, frame)
                canvas.draw()
                canvas.get_tk_widget().pack(fill=tk.BOTH, expand=True, padx=10, pady=5)

        def create_line_chart(parent, where_c, params_c):
            frame = tk.Frame(parent, bg="white", relief=tk.RAISED, bd=1)
            frame.pack(fill=tk.BOTH, padx=5, pady=10)

            tk.Label(
                frame,
                text="التطور الزمني للدورات والمشاركين",
                font=("Tajawal", 15, "bold"),
                bg="white",
                fg="#2c3e50"
            ).pack(pady=10)

            query = """
                SELECT 
                    strftime('%Y-%m', c.start_date) as month,
                    COUNT(DISTINCT c.id) as courses,
                    COUNT(p.id) as participants
                FROM courses c
                LEFT JOIN participants p ON c.id = p.course_id
                WHERE c.start_date >= date('now', '-12 months')
                    AND c.start_date IS NOT NULL
            """
            if where_c:
                query += " AND " + " AND ".join(where_c)
            query += " GROUP BY month ORDER BY month"

            cursor.execute(query, params_c)
            data = cursor.fetchall()

            if data:
                fig = Figure(figsize=(14, 5), facecolor='white')
                ax = fig.add_subplot(111)

                months = [d[0] for d in data]
                courses = [d[1] for d in data]
                participants = [d[2] for d in data]

                x = range(len(months))

                ax.plot(x, courses, marker='o', linewidth=2, label=arabic_text('الدورات'), color='#2980b9')
                ax.fill_between(x, courses, alpha=0.3, color='#2980b9')

                ax2 = ax.twinx()
                ax2.plot(x, participants, marker='s', linewidth=2, label=arabic_text('المشاركين'), color='#27ae60')

                ax.set_xlabel(arabic_text('الشهر'))
                ax.set_ylabel(arabic_text('عدد الدورات'), color='#2980b9')
                ax2.set_ylabel(arabic_text('عدد المشاركين'), color='#27ae60')

                ax.set_xticks(x[::2] if len(x) > 6 else x)
                ax.set_xticklabels([f"{m[-2:]}/{m[2:4]}" for m in months[::2 if len(months) > 6 else 1]], rotation=45)

                ax.tick_params(axis='y', labelcolor='#2980b9')
                ax2.tick_params(axis='y', labelcolor='#27ae60')
                ax.grid(True, alpha=0.3)

                lines1, labels1 = ax.get_legend_handles_labels()
                lines2, labels2 = ax2.get_legend_handles_labels()
                ax.legend(lines1 + lines2, labels1 + labels2, loc='upper left')

                plt.tight_layout()

                canvas = FigureCanvasTkAgg(fig, frame)
                canvas.draw()
                canvas.get_tk_widget().pack(fill=tk.BOTH, expand=True, padx=10, pady=5)

        def create_horizontal_chart(parent, where_c, params_c, where_p, params_p):
            frame = tk.Frame(parent, bg="white", relief=tk.RAISED, bd=1)
            frame.pack(fill=tk.BOTH, padx=5, pady=10)

            title = "أكثر 10 جهات مشاركة"
            if active_filters['category'] != "الكل":
                title += f" - {active_filters['category']}"

            tk.Label(
                frame,
                text=title,
                font=("Tajawal", 15, "bold"),
                bg="white",
                fg="#2c3e50"
            ).pack(pady=10)

            query = """
                SELECT 
                    COALESCE(p.department, 'غير محدد') as dept,
                    COUNT(*) as count
                FROM participants p
                JOIN courses c ON p.course_id = c.id
                WHERE 1=1
            """

            conditions = where_c + where_p
            params = params_c + params_p

            if conditions:
                query += " AND " + " AND ".join(conditions)

            query += " GROUP BY p.department ORDER BY count DESC LIMIT 10"

            cursor.execute(query, params)
            data = cursor.fetchall()

            if data:
                fig = Figure(figsize=(14, 5), facecolor='white')
                ax = fig.add_subplot(111)

                departments = [arabic_text(d[0][:30] + '..' if len(d[0]) > 30 else d[0]) for d in data]
                counts = [d[1] for d in data]

                y_pos = np.arange(len(departments))

                bars = ax.barh(y_pos, counts, color='#34495e', alpha=0.8)

                for i, (bar, count) in enumerate(zip(bars, counts)):
                    ax.text(bar.get_width() + max(counts) * 0.01, bar.get_y() + bar.get_height() / 2,
                            f'{count}', ha='left', va='center')

                ax.set_yticks(y_pos)
                ax.set_yticklabels(departments)
                ax.set_xlabel(arabic_text('عدد المشاركين'))
                ax.grid(True, alpha=0.3, axis='x')

                plt.tight_layout()

                canvas = FigureCanvasTkAgg(fig, frame)
                canvas.draw()
                canvas.get_tk_widget().pack(fill=tk.BOTH, expand=True, padx=10, pady=5)

        # تحميل البيانات الأولية
        update_all_data()

    def _create_courses_tab(self):
        """إنشاء تبويب الدورات مع فهرسة بسيطة"""
        courses_frame = tk.Frame(self.tab_control, bg=self.COLORS["background"])
        self.tab_control.add(courses_frame, text="الدورات")

        # متغيرات الفهرسة
        self.courses_per_page = 50
        self.current_course_page = 1
        self.total_course_pages = 1
        self.search_text = ""

        # ================== إطار البحث ==================
        search_frame = tk.Frame(courses_frame, bg="#1E3A5F", relief=tk.RAISED, bd=1)
        search_frame.pack(fill=tk.X, padx=20, pady=(20, 10))

        search_content = tk.Frame(search_frame, bg="#1E3A5F", padx=30, pady=20)
        search_content.pack(fill=tk.X)

        search_row = tk.Frame(search_content, bg="#1E3A5F")
        search_row.pack(fill=tk.X)

        tk.Label(
            search_row,
            text="🔍 البحث:",
            font=("Tajawal", 14, "bold"),
            bg="#1E3A5F",
            fg="white"
        ).pack(side=tk.LEFT, padx=(0, 10))

        search_entry = tk.Entry(
            search_row,
            font=("Tajawal", 14),
            width=35,
            bd=2,
            relief=tk.GROOVE
        )
        search_entry.pack(side=tk.LEFT, padx=(0, 10))
        search_entry.focus_set()

        tk.Label(
            search_row,
            text="(رقم الدورة أو اسم الدورة)",
            font=("Tajawal", 11),
            bg="#1E3A5F",
            fg="#B3D9FF"
        ).pack(side=tk.LEFT, padx=(0, 20))

        def perform_search():
            self.search_text = search_entry.get().strip()
            self.current_course_page = 1
            refresh_courses_table()

        search_btn = tk.Button(
            search_row,
            text="🔎 بحث",
            font=("Tajawal", 12, "bold"),
            bg="#2196F3",
            fg="white",
            padx=25,
            pady=6,
            bd=0,
            relief=tk.FLAT,
            cursor="hand2",
            command=perform_search
        )
        search_btn.pack(side=tk.LEFT, padx=5)

        show_all_btn = tk.Button(
            search_row,
            text="📋 إظهار الكل",
            font=("Tajawal", 12, "bold"),
            bg="#1565C0",
            fg="white",
            padx=25,
            pady=6,
            bd=0,
            relief=tk.FLAT,
            cursor="hand2",
            command=lambda: clear_search()
        )
        show_all_btn.pack(side=tk.LEFT, padx=5)

        def clear_search():
            search_entry.delete(0, tk.END)
            self.search_text = ""
            self.current_course_page = 1
            refresh_courses_table()

        results_label = tk.Label(
            search_row,
            text="",
            font=("Tajawal", 12, "bold"),
            bg="#1E3A5F",
            fg="#4CAF50"
        )
        results_label.pack(side=tk.LEFT, padx=20)

        search_entry.bind('<Return>', lambda e: perform_search())

        # ================== إطار الأزرار ==================
        buttons_frame = tk.Frame(courses_frame, bg=self.COLORS["surface"], height=80)
        buttons_frame.pack(fill=tk.X, padx=20, pady=(5, 0))
        buttons_frame.pack_propagate(False)

        button_container = tk.Frame(buttons_frame, bg=self.COLORS["surface"])
        button_container.pack(expand=True, fill=tk.BOTH, pady=20, padx=20)

        add_course_btn = tk.Button(
            button_container,
            text="➕ إضافة دورة جديدة",
            font=self.FONTS["text_bold"],
            bg="#2c3e50",
            fg="white",
            padx=30,
            pady=12,
            bd=0,
            relief=tk.FLAT,
            cursor="hand2",
            command=self._add_new_course
        )
        add_course_btn.pack(side=tk.LEFT, padx=5)

        edit_course_btn = tk.Button(
            button_container,
            text="✏️ تعديل الدورة",
            font=self.FONTS["text_bold"],
            bg="#2c3e50",
            fg="white",
            padx=30,
            pady=12,
            bd=0,
            relief=tk.FLAT,
            cursor="hand2",
            command=lambda: edit_selected_course()
        )
        edit_course_btn.pack(side=tk.LEFT, padx=5)

        delete_course_btn = tk.Button(
            button_container,
            text="🗑️ حذف الدورة",
            font=self.FONTS["text_bold"],
            bg="#2c3e50",
            fg="white",
            padx=30,
            pady=12,
            bd=0,
            relief=tk.FLAT,
            cursor="hand2",
            command=self._delete_course
        )
        delete_course_btn.pack(side=tk.LEFT, padx=5)

        import_participants_btn = tk.Button(
            button_container,
            text="📥 استيراد المشاركين",
            font=self.FONTS["text_bold"],
            bg="#2c3e50",
            fg="white",
            padx=30,
            pady=12,
            bd=0,
            relief=tk.FLAT,
            cursor="hand2",
            command=self._import_participants
        )
        import_participants_btn.pack(side=tk.LEFT, padx=5)

        audit_btn = tk.Button(
            button_container,
            text="✔ التدقيق والمصادقة",
            font=self.FONTS["text_bold"],
            bg="#2c3e50",
            fg="white",
            padx=30,
            pady=12,
            bd=0,
            relief=tk.FLAT,
            cursor="hand2",
            command=self._open_audit_validation
        )
        audit_btn.pack(side=tk.LEFT, padx=5)

        programs_btn = tk.Button(
            button_container,
            text="📚 إدارة البرامج والدورات",
            font=self.FONTS["text_bold"],
            bg="#2c3e50",
            fg="white",
            padx=30,
            pady=12,
            bd=0,
            relief=tk.FLAT,
            cursor="hand2",
            command=self._open_programs_management
        )
        programs_btn.pack(side=tk.LEFT, padx=5)

        # ================== إطار الجدول ==================
        table_container = tk.Frame(courses_frame, bg=self.COLORS["background"])
        table_container.pack(fill=tk.BOTH, expand=True, padx=20, pady=(10, 5))

        table_frame = tk.Frame(table_container, bg=self.COLORS["background"])
        table_frame.pack(fill=tk.BOTH, expand=True)

        columns = (
            "رقم الدورة", "اسم الدورة", "فئة الدورة", "تاريخ البداية", "تاريخ النهاية",
            "إجمالي الملتحقين", "عدد المجتازين", "عدد غير المجتازين"
        )

        self.courses_tree = ttk.Treeview(table_frame, columns=columns, show="headings", height=15)

        self.courses_tree.column("رقم الدورة", width=100, anchor="center")
        self.courses_tree.column("اسم الدورة", width=250, anchor="center")
        self.courses_tree.column("فئة الدورة", width=120, anchor="center")
        self.courses_tree.column("تاريخ البداية", width=120, anchor="center")
        self.courses_tree.column("تاريخ النهاية", width=120, anchor="center")
        self.courses_tree.column("إجمالي الملتحقين", width=130, anchor="center")
        self.courses_tree.column("عدد المجتازين", width=120, anchor="center")
        self.courses_tree.column("عدد غير المجتازين", width=140, anchor="center")

        style = ttk.Style()
        style.configure("Treeview.Heading", font=("Tajawal", 12, "bold"))
        style.configure("Treeview", font=("Tajawal", 11), rowheight=35)

        for col in columns:
            self.courses_tree.heading(col, text=col)

        scrollbar = ttk.Scrollbar(table_frame, orient="vertical", command=self.courses_tree.yview)
        self.courses_tree.configure(yscrollcommand=scrollbar.set)

        self.courses_tree.pack(side=tk.LEFT, fill=tk.BOTH, expand=True)
        scrollbar.pack(side=tk.RIGHT, fill=tk.Y)

        self.courses_tree.bind("<Double-1>", self._on_course_double_click)

        # ================== أزرار التنقل البسيطة ==================
        nav_frame = tk.Frame(courses_frame, bg=self.COLORS["background"])
        nav_frame.pack(fill=tk.X, padx=20, pady=10)

        # زر السابق
        prev_btn = tk.Button(
            nav_frame,
            text="◀ السابق",
            font=("Tajawal", 12, "bold"),
            bg="#2c3e50",
            fg="white",
            padx=25,
            pady=8,
            bd=0,
            relief=tk.FLAT,
            cursor="hand2",
            state=tk.DISABLED,
            command=lambda: navigate_page(-1)
        )
        prev_btn.pack(side=tk.LEFT, padx=10)

        # معلومات الصفحة
        page_label = tk.Label(
            nav_frame,
            text="الصفحة 1 من 1",
            font=("Tajawal", 13, "bold"),
            bg=self.COLORS["background"],
            fg="#2c3e50"
        )
        page_label.pack(side=tk.LEFT, padx=20)

        # زر التالي
        next_btn = tk.Button(
            nav_frame,
            text="التالي ▶",
            font=("Tajawal", 12, "bold"),
            bg="#2c3e50",
            fg="white",
            padx=25,
            pady=8,
            bd=0,
            relief=tk.FLAT,
            cursor="hand2",
            command=lambda: navigate_page(1)
        )
        next_btn.pack(side=tk.LEFT, padx=10)

        def navigate_page(direction):
            """التنقل بين الصفحات"""
            new_page = self.current_course_page + direction
            if 1 <= new_page <= self.total_course_pages:
                self.current_course_page = new_page
                refresh_courses_table()

        def refresh_courses_table():
            """تحديث جدول الدورات مع الفهرسة"""
            for item in self.courses_tree.get_children():
                self.courses_tree.delete(item)

            try:
                cursor = self.db_conn.cursor()

                # حساب العدد الإجمالي
                if self.search_text:
                    cursor.execute("""
                        SELECT COUNT(*) FROM courses 
                        WHERE course_code LIKE ? OR course_name LIKE ?
                    """, (f"%{self.search_text}%", f"%{self.search_text}%"))
                else:
                    cursor.execute("SELECT COUNT(*) FROM courses")

                total_courses = cursor.fetchone()[0]
                self.total_course_pages = max(1, (total_courses + self.courses_per_page - 1) // self.courses_per_page)

                offset = (self.current_course_page - 1) * self.courses_per_page

                # جلب البيانات
                if self.search_text:
                    query = """
                        SELECT id, course_code, course_name, 
                               COALESCE(course_category, '') as course_category,
                               DATE(start_date), DATE(end_date)
                        FROM courses 
                        WHERE course_code LIKE ? OR course_name LIKE ?
                        ORDER BY course_code
                        LIMIT ? OFFSET ?
                    """
                    cursor.execute(query, (f"%{self.search_text}%", f"%{self.search_text}%",
                                           self.courses_per_page, offset))
                else:
                    query = """
                        SELECT id, course_code, course_name,
                               COALESCE(course_category, '') as course_category,
                               DATE(start_date), DATE(end_date)
                        FROM courses
                        ORDER BY course_code
                        LIMIT ? OFFSET ?
                    """
                    cursor.execute(query, (self.courses_per_page, offset))

                courses = cursor.fetchall()

                for row in courses:
                    course_id = row[0]

                    # حساب إحصائيات المشاركين
                    cursor.execute("""
                        SELECT COUNT(*),
                               SUM(CASE WHEN participant_status IN ('مجتاز', 'باشر') THEN 1 ELSE 0 END),
                               SUM(CASE WHEN participant_status NOT IN ('مجتاز', 'باشر') 
                                        AND participant_status IS NOT NULL THEN 1 ELSE 0 END)
                        FROM participants WHERE course_id = ?
                    """, (course_id,))

                    stats = cursor.fetchone()

                    self.courses_tree.insert("", tk.END, values=(
                        row[1], row[2], row[3], row[4] or "", row[5] or "",
                        stats[0] or 0, stats[1] or 0, stats[2] or 0
                    ))

                # تحديث الأزرار والعداد
                page_label.config(text=f"الصفحة {self.current_course_page} من {self.total_course_pages}")
                prev_btn.config(state=tk.NORMAL if self.current_course_page > 1 else tk.DISABLED)
                next_btn.config(state=tk.NORMAL if self.current_course_page < self.total_course_pages else tk.DISABLED)

                if self.search_text:
                    results_label.config(text=f"النتائج: {total_courses} دورة")
                else:
                    results_label.config(text=f"الإجمالي: {total_courses} دورة")

            except Exception as e:
                messagebox.showerror("خطأ", f"حدث خطأ في تحديث الجدول:\n{str(e)}")

        def edit_selected_course():
            """تعديل الدورة المحددة"""
            selected = self.courses_tree.selection()
            if not selected:
                messagebox.showwarning("تنبيه", "الرجاء اختيار دورة للتعديل")
                return

            item = self.courses_tree.item(selected[0])
            course_data = item["values"]

            if not course_data or not course_data[0]:
                return

            self._edit_course_dialog(str(course_data[0]), refresh_courses_table)

        refresh_courses_table()

    # ================== نظام التصدير المحسن لصفحة التقارير ==================

    def _create_reports_tab(self):
        """إنشاء تبويب التقارير مع نظام التصدير المحسن"""
        reports_frame = tk.Frame(self.tab_control, bg=self.COLORS["background"])
        self.tab_control.add(reports_frame, text="التقارير")

        # شريط العنوان
        header = tk.Frame(reports_frame, bg="#1E3A5F", height=80)
        header.pack(fill=tk.X)
        header.pack_propagate(False)

        header_content = tk.Frame(header, bg="#1E3A5F")
        header_content.pack(expand=True, fill=tk.BOTH)

        tk.Label(
            header_content,
            text="التقارير والإحصائيات",
            font=self.FONTS["large_title"],
            bg="#1E3A5F",
            fg="white"
        ).pack(expand=True)

        # ================== قسم أزرار التصدير ==================
        export_section = tk.Frame(reports_frame, bg="white", relief=tk.RAISED, bd=1)
        export_section.pack(fill=tk.X, padx=30, pady=30)

        export_content = tk.Frame(export_section, bg="white", padx=40, pady=30)
        export_content.pack(fill=tk.BOTH)

        # عنوان القسم
        tk.Label(
            export_content,
            text="تصدير البيانات",
            font=("Tajawal", 20, "bold"),
            bg="white",
            fg="#263238"
        ).pack(anchor="e", pady=(0, 20))

        # إطار الأزرار الرئيسية
        main_buttons_frame = tk.Frame(export_content, bg="white")
        main_buttons_frame.pack(pady=10)

        # ================== دالة تحديد حالة الدورة ==================
        def get_course_status(start_date, end_date):
            """تحديد حالة الدورة بناءً على التواريخ"""
            import datetime

            try:
                today = datetime.datetime.now().date()

                # تحويل التواريخ إلى datetime.date
                if isinstance(start_date, str):
                    if start_date and start_date != '':
                        start_date = datetime.datetime.strptime(start_date.split()[0], '%Y-%m-%d').date()
                    else:
                        return "غير محدد"

                if isinstance(end_date, str):
                    if end_date and end_date != '':
                        end_date = datetime.datetime.strptime(end_date.split()[0], '%Y-%m-%d').date()
                    else:
                        return "غير محدد"

                # تحديد الحالة
                if end_date < today:
                    return "منتهية"
                elif start_date > today:
                    return "لم تبدأ"
                else:
                    return "جارية"

            except:
                return "غير محدد"

        # ================== دالة تصدير جميع البيانات المحسنة ==================
        def export_all_data():
            """تصدير جميع بيانات الدورات والمشاركين مع تقرير الجهات"""
            try:
                import pandas as pd
                from tkinter import filedialog
                import datetime
                import os
                import subprocess
                import platform

                # اختيار مكان الحفظ
                file_path = filedialog.asksaveasfilename(
                    defaultextension=".xlsx",
                    filetypes=[("Excel files", "*.xlsx")],
                    initialfile=f"تقرير_كامل_{datetime.datetime.now().strftime('%Y%m%d_%H%M%S')}.xlsx"
                )

                if not file_path:
                    return

                cursor = self.db_conn.cursor()

                # جلب بيانات الدورات المحسنة
                cursor.execute("""
                    SELECT 
                        c.course_code,
                        c.course_name,
                        c.course_category,
                        DATE(c.start_date) as start_date,
                        DATE(c.end_date) as end_date,
                        c.gender,
                        c.requirements,
                        COUNT(DISTINCT p.id) as participants_count,
                        SUM(CASE WHEN p.participant_status IN ('مجتاز', 'باشر') THEN 1 ELSE 0 END) as passed_count,
                        SUM(CASE WHEN p.participant_status IS NOT NULL 
                            AND p.participant_status NOT IN ('مجتاز', 'باشر') THEN 1 ELSE 0 END) as not_passed_count
                    FROM courses c
                    LEFT JOIN participants p ON c.id = p.course_id
                    GROUP BY c.id, c.course_code, c.course_name, c.course_category, 
                             c.start_date, c.end_date, c.gender, c.requirements
                    ORDER BY c.course_code
                """)

                courses_data = cursor.fetchall()

                # تحديد حالة الدورة
                def get_course_status(start_date, end_date):
                    """تحديد حالة الدورة بناءً على التواريخ"""
                    try:
                        today = datetime.datetime.now().date()

                        if isinstance(start_date, str):
                            if start_date and start_date != '':
                                start_date = datetime.datetime.strptime(start_date.split()[0], '%Y-%m-%d').date()
                            else:
                                return "غير محدد"

                        if isinstance(end_date, str):
                            if end_date and end_date != '':
                                end_date = datetime.datetime.strptime(end_date.split()[0], '%Y-%m-%d').date()
                            else:
                                return "غير محدد"

                        if end_date < today:
                            return "منتهية"
                        elif start_date > today:
                            return "لم تبدأ"
                        else:
                            return "جارية"
                    except:
                        return "غير محدد"

                # تحضير بيانات الدورات
                courses_list = []
                courses_without_participants = 0

                for row in courses_data:
                    course_status = get_course_status(row[3], row[4])
                    participants_count = row[7] if row[7] else 0
                    passed_count = row[8] if row[8] else 0
                    not_passed_count = row[9] if row[9] else 0

                    # إذا لم يكن هناك مشاركين، لا نحسب غير المجتازين
                    if participants_count == 0:
                        courses_without_participants += 1
                        not_passed_count = 0

                    courses_list.append({
                        'رقم الدورة': row[0],
                        'اسم الدورة': row[1],
                        'فئة الدورة': row[2] if row[2] else '',
                        'تاريخ البداية': row[3] if row[3] else '',
                        'تاريخ النهاية': row[4] if row[4] else '',
                        'الجنس': row[5] if row[5] else '',
                        'الاشتراطات': row[6] if row[6] else '',
                        'حالة الدورة': course_status,
                        'عدد المشاركين': participants_count,
                        'عدد المجتازين': passed_count,
                        'عدد غير المجتازين': not_passed_count
                    })

                courses_df = pd.DataFrame(courses_list)

                # جلب بيانات المشاركين
                cursor.execute("""
                    SELECT 
                        c.course_code,
                        c.course_name,
                        p.participant_name,
                        p.national_id,
                        p.rank,
                        p.mobile,
                        p.department,
                        p.sub_department,
                        p.work_nature,
                        p.sector,
                        p.email,
                        p.training_location,
                        p.course_status,
                        p.participant_status,
                        p.cancellation_reason
                    FROM participants p
                    JOIN courses c ON p.course_id = c.id
                    ORDER BY c.course_code, p.participant_name
                """)

                participants_data = cursor.fetchall()

                participants_list = []
                for row in participants_data:
                    participants_list.append({
                        'رقم الدورة': row[0],
                        'اسم الدورة': row[1],
                        'اسم المشارك': row[2],
                        'رقم الهوية': row[3],
                        'الرتبة': row[4] if row[4] else '',
                        'رقم الجوال': row[5] if row[5] else '',
                        'الجهة': row[6] if row[6] else '',
                        'الجهة الفرعية': row[7] if row[7] else '',
                        'طبيعة العمل': row[8] if row[8] else '',
                        'القطاع': row[9] if row[9] else '',
                        'البريد الإلكتروني': row[10] if row[10] else '',
                        'مقر التدريب': row[11] if row[11] else '',
                        'حالة الدورة': row[12] if row[12] else '',
                        'حالة المشارك': row[13] if row[13] else '',
                        'سبب الإلغاء': row[14] if row[14] else ''
                    })

                participants_df = pd.DataFrame(participants_list)

                # ================== إعداد تقرير الجهات ==================
                cursor.execute("""
                    SELECT 
                        p.department as 'الجهة',
                        p.rank as 'الفئة',
                        COUNT(DISTINCT p.id) as 'إجمالي المستفيدين',
                        COUNT(DISTINCT CASE WHEN p.participant_status IN ('مجتاز', 'باشر') THEN p.id END) as 'عدد المجتازين',
                        COUNT(DISTINCT CASE WHEN p.participant_status NOT IN ('مجتاز', 'باشر') 
                            AND p.participant_status IS NOT NULL THEN p.id END) as 'عدد غير المجتازين',
                        COUNT(DISTINCT c.id) as 'عدد الدورات',
                        ROUND(CAST(COUNT(DISTINCT CASE WHEN p.participant_status IN ('مجتاز', 'باشر') THEN p.id END) AS FLOAT) 
                            / NULLIF(COUNT(DISTINCT p.id), 0) * 100, 2) as 'نسبة النجاح'
                    FROM participants p
                    JOIN courses c ON p.course_id = c.id
                    WHERE p.department IS NOT NULL AND p.department != ''
                    GROUP BY p.department, p.rank
                    ORDER BY p.department, p.rank
                """)

                departments_by_rank = cursor.fetchall()

                # تقرير الجهات حسب الفئات
                dept_rank_list = []
                for row in departments_by_rank:
                    dept_rank_list.append({
                        'الجهة': row[0],
                        'الفئة': row[1] if row[1] else 'غير محدد',
                        'إجمالي المستفيدين': row[2],
                        'عدد المجتازين': row[3],
                        'عدد غير المجتازين': row[4],
                        'عدد الدورات': row[5],
                        'نسبة النجاح %': f"{row[6]:.1f}" if row[6] else "0.0"
                    })

                dept_rank_df = pd.DataFrame(dept_rank_list)

                # تقرير إجمالي الجهات
                cursor.execute("""
                    SELECT 
                        p.department as 'الجهة',
                        COUNT(DISTINCT p.id) as 'إجمالي المستفيدين',
                        COUNT(DISTINCT CASE WHEN p.participant_status IN ('مجتاز', 'باشر') THEN p.id END) as 'عدد المجتازين',
                        COUNT(DISTINCT CASE WHEN p.participant_status NOT IN ('مجتاز', 'باشر') 
                            AND p.participant_status IS NOT NULL THEN p.id END) as 'عدد غير المجتازين',
                        COUNT(DISTINCT c.id) as 'عدد الدورات',
                        COUNT(DISTINCT p.rank) as 'عدد الفئات',
                        ROUND(CAST(COUNT(DISTINCT CASE WHEN p.participant_status IN ('مجتاز', 'باشر') THEN p.id END) AS FLOAT) 
                            / NULLIF(COUNT(DISTINCT p.id), 0) * 100, 2) as 'نسبة النجاح الإجمالية'
                    FROM participants p
                    JOIN courses c ON p.course_id = c.id
                    WHERE p.department IS NOT NULL AND p.department != ''
                    GROUP BY p.department
                    ORDER BY COUNT(DISTINCT p.id) DESC
                """)

                departments_total = cursor.fetchall()

                dept_total_list = []
                for row in departments_total:
                    dept_total_list.append({
                        'الجهة': row[0],
                        'إجمالي المستفيدين': row[1],
                        'عدد المجتازين': row[2],
                        'عدد غير المجتازين': row[3],
                        'عدد الدورات': row[4],
                        'عدد الفئات': row[5],
                        'نسبة النجاح %': f"{row[6]:.1f}" if row[6] else "0.0"
                    })

                dept_total_df = pd.DataFrame(dept_total_list)

                # إحصائيات محسنة
                total_courses = len(courses_df)
                total_participants = len(participants_df)
                total_passed = participants_df[participants_df['حالة المشارك'].isin(['مجتاز', 'باشر'])].shape[0]
                total_not_passed = participants_df[
                    ~participants_df['حالة المشارك'].isin(['مجتاز', 'باشر']) &
                    participants_df['حالة المشارك'].notna()
                    ].shape[0]

                # إحصائيات حالة الدورات
                finished_courses = courses_df[courses_df['حالة الدورة'] == 'منتهية'].shape[0]
                ongoing_courses = courses_df[courses_df['حالة الدورة'] == 'جارية'].shape[0]
                future_courses = courses_df[courses_df['حالة الدورة'] == 'لم تبدأ'].shape[0]

                stats_data = {
                    'الإحصائية': [
                        'إجمالي عدد الدورات',
                        'الدورات المنتهية',
                        'الدورات الجارية',
                        'الدورات التي لم تبدأ',
                        'الدورات بدون مشاركين',
                        'إجمالي عدد المشاركين',
                        'إجمالي عدد المجتازين',
                        'إجمالي عدد غير المجتازين',
                        'متوسط المشاركين في الدورة',
                        'نسبة النجاح الإجمالية',
                        'عدد الجهات المستفيدة',
                        'أكثر جهة استفادة'
                    ],
                    'القيمة': [
                        total_courses,
                        finished_courses,
                        ongoing_courses,
                        future_courses,
                        courses_without_participants,
                        total_participants,
                        total_passed,
                        total_not_passed,
                        f"{total_participants / total_courses:.1f}" if total_courses > 0 else "0",
                        f"{(total_passed / total_participants * 100):.1f}%" if total_participants > 0 else "0%",
                        len(dept_total_df),
                        dept_total_df.iloc[0]['الجهة'] if len(dept_total_df) > 0 else "—"
                    ]
                }
                stats_df = pd.DataFrame(stats_data)

                # الدورات بدون مشاركين
                empty_courses_df = courses_df[courses_df['عدد المشاركين'] == 0][
                    ['رقم الدورة', 'اسم الدورة', 'حالة الدورة', 'تاريخ البداية', 'تاريخ النهاية']
                ]

                # إنشاء ملف Excel مع التنسيق
                with pd.ExcelWriter(file_path, engine='openpyxl') as writer:
                    # ورقة الإحصائيات (أولاً)
                    stats_df.to_excel(writer, sheet_name='الإحصائيات', index=False)

                    # ورقة الدورات
                    courses_df.to_excel(writer, sheet_name='الدورات', index=False)

                    # ورقة المشاركين
                    if not participants_df.empty:
                        participants_df.to_excel(writer, sheet_name='المشاركين', index=False)

                    # ورقة تفاصيل الجهات
                    if not dept_rank_df.empty:
                        dept_rank_df.to_excel(writer, sheet_name='تفاصيل الجهات', index=False)

                    # ورقة إجمالي الجهات
                    if not dept_total_df.empty:
                        dept_total_df.to_excel(writer, sheet_name='إجمالي الجهات', index=False)

                    # ورقة الدورات بدون مشاركين
                    if not empty_courses_df.empty:
                        empty_courses_df.to_excel(writer, sheet_name='دورات بدون مشاركين', index=False)

                    # تنسيق الأوراق
                    workbook = writer.book

                    for sheet_name in workbook.sheetnames:
                        worksheet = workbook[sheet_name]

                        # اتجاه من اليمين لليسار
                        worksheet.sheet_view.rightToLeft = True

                        # تنسيق الخلايا
                        from openpyxl.styles import Font, Alignment, PatternFill, Border, Side

                        # تنسيق الرأس
                        header_font = Font(name='Tajawal', size=12, bold=True, color='FFFFFF')
                        header_fill = PatternFill(start_color='1E3A5F', end_color='1E3A5F', fill_type='solid')
                        header_alignment = Alignment(horizontal='center', vertical='center')

                        # تنسيق البيانات
                        data_font = Font(name='Tajawal', size=11)
                        data_alignment = Alignment(horizontal='center', vertical='center', wrap_text=True)

                        # حدود
                        thin_border = Border(
                            left=Side(style='thin'),
                            right=Side(style='thin'),
                            top=Side(style='thin'),
                            bottom=Side(style='thin')
                        )

                        # تطبيق التنسيق على الرأس
                        for cell in worksheet[1]:
                            cell.font = header_font
                            cell.fill = header_fill
                            cell.alignment = header_alignment
                            cell.border = thin_border

                        # تطبيق التنسيق على البيانات
                        for row in worksheet.iter_rows(min_row=2):
                            for cell in row:
                                cell.font = data_font
                                cell.alignment = data_alignment
                                cell.border = thin_border

                        # تنسيق خاص لورقة الإحصائيات
                        if sheet_name == 'الإحصائيات':
                            # تلوين الصفوف بالتناوب
                            for i, row in enumerate(worksheet.iter_rows(min_row=2), start=2):
                                if i % 2 == 0:
                                    for cell in row:
                                        cell.fill = PatternFill(start_color='F5F5F5', end_color='F5F5F5',
                                                                fill_type='solid')

                        # تنسيق خاص لورقة الجهات
                        if 'الجهات' in sheet_name:
                            # تلوين النسب حسب القيمة
                            for row in worksheet.iter_rows(min_row=2):
                                for cell in row:
                                    if cell.column_letter == 'G':  # عمود النسبة
                                        try:
                                            value = float(str(cell.value).replace('%', ''))
                                            if value >= 80:
                                                cell.font = Font(name='Tajawal', size=11, bold=True, color='2E7D32')
                                            elif value >= 60:
                                                cell.font = Font(name='Tajawal', size=11, bold=True, color='FF9800')
                                            else:
                                                cell.font = Font(name='Tajawal', size=11, bold=True, color='D32F2F')
                                        except:
                                            pass

                        # ضبط عرض الأعمدة تلقائياً
                        for column in worksheet.columns:
                            max_length = 0
                            column_letter = column[0].column_letter

                            for cell in column:
                                try:
                                    if len(str(cell.value)) > max_length:
                                        max_length = len(str(cell.value))
                                except:
                                    pass

                            adjusted_width = min(max_length + 5, 50)
                            worksheet.column_dimensions[column_letter].width = adjusted_width

                        # تجميد الصف الأول
                        worksheet.freeze_panes = 'A2'

                # عرض رسالة النجاح
                messagebox.showinfo(
                    "نجاح",
                    f"تم تصدير البيانات بنجاح\n\n"
                    f"📊 إجمالي الدورات: {total_courses}\n"
                    f"👥 إجمالي المشاركين: {total_participants}\n"
                    f"🏢 عدد الجهات المستفيدة: {len(dept_total_df)}\n"
                    f"⚠️ دورات بدون مشاركين: {courses_without_participants}\n\n"
                    f"سيتم فتح الملف الآن..."
                )

                # فتح الملف تلقائياً
                try:
                    if platform.system() == 'Windows':
                        os.startfile(file_path)
                    elif platform.system() == 'Darwin':  # macOS
                        subprocess.call(['open', file_path])
                    else:  # Linux
                        subprocess.call(['xdg-open', file_path])
                except Exception as e:
                    print(f"لم نتمكن من فتح الملف تلقائياً: {e}")

            except ImportError:
                messagebox.showerror("خطأ", "يجب تثبيت مكتبة pandas و openpyxl\npip install pandas openpyxl")
            except Exception as e:
                messagebox.showerror("خطأ", f"حدث خطأ أثناء التصدير:\n{str(e)}")

        # ================== دالة التصدير بحسب الفترة المحسنة ==================
        def export_by_period():
            """فتح نافذة التصدير بحسب الفترة مع خيارات متعددة"""
            from tkinter import ttk
            import datetime

            period_window = tk.Toplevel(self)
            period_window.title("تصدير بحسب الفترة")
            period_window.geometry("700x600")
            period_window.configure(bg="#f8f9fa")
            period_window.transient(self)
            period_window.grab_set()

            # توسيط النافذة
            period_window.update_idletasks()
            x = (period_window.winfo_screenwidth() - 700) // 2
            y = (period_window.winfo_screenheight() - 600) // 2
            period_window.geometry(f"700x600+{x}+{y}")

            # الشريط العلوي
            header = tk.Frame(period_window, bg="#37474f", height=80)
            header.pack(fill=tk.X)
            header.pack_propagate(False)

            tk.Label(
                header,
                text="تصدير البيانات بحسب الفترة",
                font=("Tajawal", 18, "bold"),
                bg="#37474f",
                fg="white"
            ).pack(expand=True)

            # المحتوى
            content = tk.Frame(period_window, bg="white")
            content.pack(fill=tk.BOTH, expand=True, padx=20, pady=20)

            # ================== خيارات الربع السنوي المتعددة ==================
            quarter_frame = tk.LabelFrame(
                content,
                text="تصدير بحسب الأرباع السنوية",
                font=("Tajawal", 14, "bold"),
                bg="white",
                fg="#37474f",
                padx=20,
                pady=20
            )
            quarter_frame.pack(fill=tk.X, pady=10)

            # السنة
            year_frame = tk.Frame(quarter_frame, bg="white")
            year_frame.pack(fill=tk.X, pady=10)

            tk.Label(
                year_frame,
                text="السنة:",
                font=("Tajawal", 12),
                bg="white"
            ).pack(side=tk.RIGHT, padx=10)

            current_year = datetime.datetime.now().year
            years = list(range(current_year - 5, current_year + 2))

            year_var = tk.StringVar(value=str(current_year))
            year_combo = ttk.Combobox(
                year_frame,
                textvariable=year_var,
                values=years,
                state="readonly",
                width=10,
                font=("Tajawal", 11)
            )
            year_combo.pack(side=tk.RIGHT, padx=10)

            # الأرباع - خيارات متعددة
            quarters_frame = tk.Frame(quarter_frame, bg="white")
            quarters_frame.pack(fill=tk.X, pady=10)

            tk.Label(
                quarters_frame,
                text="اختر الأرباع:",
                font=("Tajawal", 12),
                bg="white"
            ).pack(side=tk.RIGHT, padx=10)

            quarter_vars = {
                "الأول": tk.BooleanVar(value=False),
                "الثاني": tk.BooleanVar(value=False),
                "الثالث": tk.BooleanVar(value=False),
                "الرابع": tk.BooleanVar(value=False)
            }

            for quarter, var in quarter_vars.items():
                tk.Checkbutton(
                    quarters_frame,
                    text=f"الربع {quarter}",
                    variable=var,
                    font=("Tajawal", 11),
                    bg="white"
                ).pack(side=tk.RIGHT, padx=10)

            # زر تحديد الكل
            def select_all_quarters():
                for var in quarter_vars.values():
                    var.set(True)

            def deselect_all_quarters():
                for var in quarter_vars.values():
                    var.set(False)

            select_buttons_frame = tk.Frame(quarter_frame, bg="white")
            select_buttons_frame.pack(fill=tk.X, pady=5)

            tk.Button(
                select_buttons_frame,
                text="تحديد الكل",
                font=("Tajawal", 11),
                bg="#90a4ae",
                fg="white",
                padx=15,
                pady=5,
                bd=0,
                relief=tk.FLAT,
                cursor="hand2",
                command=select_all_quarters
            ).pack(side=tk.RIGHT, padx=5)

            tk.Button(
                select_buttons_frame,
                text="إلغاء التحديد",
                font=("Tajawal", 11),
                bg="#90a4ae",
                fg="white",
                padx=15,
                pady=5,
                bd=0,
                relief=tk.FLAT,
                cursor="hand2",
                command=deselect_all_quarters
            ).pack(side=tk.RIGHT, padx=5)

            def export_quarters():
                """تصدير الأرباع المحددة"""
                selected_quarters = [q for q, var in quarter_vars.items() if var.get()]

                if not selected_quarters:
                    messagebox.showwarning("تنبيه", "يرجى اختيار ربع واحد على الأقل")
                    return

                year = int(year_var.get())

                # تحديد التواريخ للأرباع المحددة
                all_data = []
                quarter_map = {
                    "الأول": (f"{year}-01-01", f"{year}-03-31"),
                    "الثاني": (f"{year}-04-01", f"{year}-06-30"),
                    "الثالث": (f"{year}-07-01", f"{year}-09-30"),
                    "الرابع": (f"{year}-10-01", f"{year}-12-31")
                }

                # تحديد أوسع نطاق للتواريخ
                start_dates = []
                end_dates = []

                for quarter in selected_quarters:
                    start, end = quarter_map[quarter]
                    start_dates.append(start)
                    end_dates.append(end)

                overall_start = min(start_dates)
                overall_end = max(end_dates)

                quarters_text = "_".join(
                    [f"Q{i + 1}" for i, q in enumerate(["الأول", "الثاني", "الثالث", "الرابع"]) if
                     q in selected_quarters])

                export_data_by_dates(overall_start, overall_end, f"الأرباع_{quarters_text}_{year}", selected_quarters,
                                     year)
                period_window.destroy()

            tk.Button(
                quarter_frame,
                text="تصدير الأرباع المحددة",
                font=("Tajawal", 12, "bold"),
                bg="#455a64",
                fg="white",
                padx=25,
                pady=10,
                bd=0,
                relief=tk.FLAT,
                cursor="hand2",
                command=export_quarters
            ).pack(pady=10)

            # ================== خيارات الفترة المخصصة ==================
            custom_frame = tk.LabelFrame(
                content,
                text="تصدير بحسب فترة مخصصة",
                font=("Tajawal", 14, "bold"),
                bg="white",
                fg="#37474f",
                padx=20,
                pady=20
            )
            custom_frame.pack(fill=tk.X, pady=10)

            # تاريخ البداية
            start_frame = tk.Frame(custom_frame, bg="white")
            start_frame.pack(fill=tk.X, pady=10)

            tk.Label(
                start_frame,
                text="من تاريخ:",
                font=("Tajawal", 12),
                bg="white"
            ).pack(side=tk.RIGHT, padx=10)

            start_date_entry = tk.Entry(
                start_frame,
                font=("Tajawal", 11),
                width=15
            )
            start_date_entry.pack(side=tk.RIGHT, padx=10)
            start_date_entry.insert(0, datetime.datetime.now().strftime("%Y-%m-%d"))

            # تاريخ النهاية
            end_frame = tk.Frame(custom_frame, bg="white")
            end_frame.pack(fill=tk.X, pady=10)

            tk.Label(
                end_frame,
                text="إلى تاريخ:",
                font=("Tajawal", 12),
                bg="white"
            ).pack(side=tk.RIGHT, padx=10)

            end_date_entry = tk.Entry(
                end_frame,
                font=("Tajawal", 11),
                width=15
            )
            end_date_entry.pack(side=tk.RIGHT, padx=10)
            end_date_entry.insert(0, datetime.datetime.now().strftime("%Y-%m-%d"))

            def export_custom():
                """تصدير بيانات الفترة المخصصة"""
                start = start_date_entry.get()
                end = end_date_entry.get()

                try:
                    # التحقق من صحة التواريخ
                    datetime.datetime.strptime(start, "%Y-%m-%d")
                    datetime.datetime.strptime(end, "%Y-%m-%d")

                    export_data_by_dates(start, end, f"فترة_{start}_إلى_{end}")
                    period_window.destroy()
                except ValueError:
                    messagebox.showerror("خطأ", "صيغة التاريخ غير صحيحة\nاستخدم: YYYY-MM-DD")

            tk.Button(
                custom_frame,
                text="تصدير الفترة المحددة",
                font=("Tajawal", 12, "bold"),
                bg="#455a64",
                fg="white",
                padx=25,
                pady=10,
                bd=0,
                relief=tk.FLAT,
                cursor="hand2",
                command=export_custom
            ).pack(pady=10)

            # زر الإغلاق
            tk.Button(
                content,
                text="إلغاء",
                font=("Tajawal", 12, "bold"),
                bg="#90a4ae",
                fg="white",
                padx=30,
                pady=10,
                bd=0,
                relief=tk.FLAT,
                cursor="hand2",
                command=period_window.destroy
            ).pack(pady=20)

        # ================== دالة التصدير بحسب التواريخ المحسنة ==================
        def export_data_by_dates(start_date, end_date, file_prefix="", quarters=None, year=None):
            """تصدير البيانات ضمن فترة زمنية محددة"""
            try:
                import pandas as pd
                from tkinter import filedialog
                import datetime
                import os
                import subprocess
                import platform

                # اختيار مكان الحفظ
                file_path = filedialog.asksaveasfilename(
                    defaultextension=".xlsx",
                    filetypes=[("Excel files", "*.xlsx")],
                    initialfile=f"تقرير_{file_prefix}_{datetime.datetime.now().strftime('%Y%m%d')}.xlsx"
                )

                if not file_path:
                    return

                cursor = self.db_conn.cursor()

                # جلب بيانات الدورات ضمن الفترة
                cursor.execute("""
                    SELECT 
                        c.course_code,
                        c.course_name,
                        c.course_category,
                        DATE(c.start_date) as start_date,
                        DATE(c.end_date) as end_date,
                        c.gender,
                        c.requirements,
                        COUNT(p.id) as participants_count,
                        SUM(CASE WHEN p.participant_status IN ('مجتاز', 'باشر') THEN 1 ELSE 0 END) as passed_count
                    FROM courses c
                    LEFT JOIN participants p ON c.id = p.course_id
                    WHERE DATE(c.start_date) BETWEEN ? AND ?
                       OR DATE(c.end_date) BETWEEN ? AND ?
                       OR (DATE(c.start_date) <= ? AND DATE(c.end_date) >= ?)
                    GROUP BY c.id
                    ORDER BY c.start_date, c.course_code
                """, (start_date, end_date, start_date, end_date, start_date, end_date))

                courses_data = cursor.fetchall()

                if not courses_data:
                    messagebox.showwarning("تنبيه", f"لا توجد دورات في الفترة المحددة\n{start_date} إلى {end_date}")
                    return

                # تحضير البيانات
                courses_list = []
                for row in courses_data:
                    course_status = get_course_status(row[3], row[4])

                    courses_list.append({
                        'رقم الدورة': row[0],
                        'اسم الدورة': row[1],
                        'فئة الدورة': row[2] if row[2] else '',
                        'تاريخ البداية': row[3] if row[3] else '',
                        'تاريخ النهاية': row[4] if row[4] else '',
                        'الجنس': row[5] if row[5] else '',
                        'الاشتراطات': row[6] if row[6] else '',
                        'حالة الدورة': course_status,
                        'عدد المشاركين': row[7] if row[7] else 0,
                        'عدد المجتازين': row[8] if row[8] else 0
                    })

                courses_df = pd.DataFrame(courses_list)

                # حفظ وتنسيق الملف
                with pd.ExcelWriter(file_path, engine='openpyxl') as writer:
                    # إذا كانت أرباع متعددة، أنشئ ورقة لكل ربع
                    if quarters and year:
                        quarter_map = {
                            "الأول": (f"{year}-01-01", f"{year}-03-31"),
                            "الثاني": (f"{year}-04-01", f"{year}-06-30"),
                            "الثالث": (f"{year}-07-01", f"{year}-09-30"),
                            "الرابع": (f"{year}-10-01", f"{year}-12-31")
                        }

                        for quarter in quarters:
                            q_start, q_end = quarter_map[quarter]
                            quarter_df = courses_df[
                                (courses_df['تاريخ البداية'] >= q_start) &
                                (courses_df['تاريخ البداية'] <= q_end)
                                ]

                            if not quarter_df.empty:
                                quarter_df.to_excel(writer, sheet_name=f'الربع {quarter}', index=False)

                    # ورقة لجميع البيانات
                    courses_df.to_excel(writer, sheet_name='جميع الدورات', index=False)

                    # إضافة ورقة الإحصائيات
                    stats_data = {
                        'الإحصائية': [
                            'الفترة',
                            'إجمالي الدورات',
                            'الدورات المنتهية',
                            'الدورات الجارية',
                            'الدورات التي لم تبدأ',
                            'إجمالي المشاركين',
                            'إجمالي المجتازين'
                        ],
                        'القيمة': [
                            f"{start_date} إلى {end_date}",
                            len(courses_df),
                            courses_df[courses_df['حالة الدورة'] == 'منتهية'].shape[0],
                            courses_df[courses_df['حالة الدورة'] == 'جارية'].shape[0],
                            courses_df[courses_df['حالة الدورة'] == 'لم تبدأ'].shape[0],
                            courses_df['عدد المشاركين'].sum(),
                            courses_df['عدد المجتازين'].sum()
                        ]
                    }
                    stats_df = pd.DataFrame(stats_data)
                    stats_df.to_excel(writer, sheet_name='الإحصائيات', index=False)

                    # التنسيق
                    workbook = writer.book
                    for sheet_name in workbook.sheetnames:
                        worksheet = workbook[sheet_name]
                        worksheet.sheet_view.rightToLeft = True

                        from openpyxl.styles import Font, Alignment, PatternFill, Border, Side

                        header_font = Font(name='Tajawal', size=12, bold=True, color='FFFFFF')
                        header_fill = PatternFill(start_color='37474f', end_color='37474f', fill_type='solid')
                        header_alignment = Alignment(horizontal='center', vertical='center')

                        for cell in worksheet[1]:
                            cell.font = header_font
                            cell.fill = header_fill
                            cell.alignment = header_alignment

                        # ضبط عرض الأعمدة
                        for column in worksheet.columns:
                            max_length = 0
                            column_letter = column[0].column_letter
                            for cell in column:
                                try:
                                    if len(str(cell.value)) > max_length:
                                        max_length = len(str(cell.value))
                                except:
                                    pass
                            adjusted_width = min(max_length + 5, 50)
                            worksheet.column_dimensions[column_letter].width = adjusted_width

                messagebox.showinfo("نجاح", f"تم تصدير {len(courses_df)} دورة بنجاح")

                # فتح الملف
                try:
                    if platform.system() == 'Windows':
                        os.startfile(file_path)
                    elif platform.system() == 'Darwin':
                        subprocess.call(['open', file_path])
                    else:
                        subprocess.call(['xdg-open', file_path])
                except:
                    pass

            except Exception as e:
                messagebox.showerror("خطأ", f"حدث خطأ أثناء التصدير:\n{str(e)}")

        # زر تصدير جميع البيانات
        export_all_btn = tk.Button(
            main_buttons_frame,
            text="📥 تصدير جميع البيانات",
            font=("Tajawal", 14, "bold"),
            bg="#37474f",
            fg="white",
            padx=40,
            pady=15,
            bd=0,
            relief=tk.FLAT,
            cursor="hand2",
            command=export_all_data
        )
        export_all_btn.pack(side=tk.RIGHT, padx=10)

        # زر التصدير بحسب الفترة
        export_period_btn = tk.Button(
            main_buttons_frame,
            text="📅 تصدير بحسب الفترة",
            font=("Tajawal", 14, "bold"),
            bg="#546e7a",
            fg="white",
            padx=40,
            pady=15,
            bd=0,
            relief=tk.FLAT,
            cursor="hand2",
            command=export_by_period
        )
        export_period_btn.pack(side=tk.RIGHT, padx=10)

        # ملاحظة
        tk.Label(
            export_content,
            text="ملاحظة: سيتم فتح ملف Excel تلقائياً بعد التصدير",
            font=("Tajawal", 11),
            bg="white",
            fg="#90a4ae"
        ).pack(pady=(20, 0))


    def _load_courses(self):
        """تحميل قائمة الدورات مع حساب صحيح للمجتازين وغير المجتازين"""
        # مسح البيانات الحالية من الجدول
        for item in self.courses_tree.get_children():
            self.courses_tree.delete(item)

        try:
            cursor = self.db_conn.cursor()

            # استعلام SQL محسّن - يستخدم DATE() لإزالة الوقت من التواريخ
            cursor.execute("""
                SELECT 
                    id,
                    course_code,
                    course_name,
                    COALESCE(course_category, '') as course_category,
                    DATE(start_date) as start_date,
                    DATE(end_date) as end_date
                FROM courses 
                ORDER BY course_code
            """)

            courses = cursor.fetchall()

            for row in courses:
                # استخراج بيانات الدورة
                course_id = row[0]
                course_code = row[1]
                course_name = row[2]
                course_category = row[3]
                start_date = row[4] if row[4] else ""
                end_date = row[5] if row[5] else ""

                # حساب إحصائيات المشاركين للدورة بشكل تفصيلي
                cursor.execute("""
                    SELECT 
                        COUNT(*) as total,
                        SUM(CASE 
                            WHEN participant_status = 'مجتاز' OR participant_status = 'باشر' 
                            THEN 1 
                            ELSE 0 
                        END) as passed,
                        SUM(CASE 
                            WHEN participant_status = 'إلغاء دورة' 
                            THEN 1 
                            ELSE 0 
                        END) as cancelled,
                        SUM(CASE 
                            WHEN participant_status = 'لم يباشر' 
                            THEN 1 
                            ELSE 0 
                        END) as not_started,
                        SUM(CASE 
                            WHEN participant_status = 'مرفوض أخرى' OR participant_status = 'مرفوض' 
                            THEN 1 
                            ELSE 0 
                        END) as rejected_other,
                        SUM(CASE 
                            WHEN participant_status NOT IN ('مجتاز', 'باشر') 
                                 OR participant_status IS NULL 
                            THEN 1 
                            ELSE 0 
                        END) as total_not_passed
                    FROM participants 
                    WHERE course_id = ?
                """, (course_id,))

                stats = cursor.fetchone()

                # استخراج الإحصائيات
                total_participants = stats[0] if stats[0] else 0
                passed_count = stats[1] if stats[1] else 0
                cancelled_count = stats[2] if stats[2] else 0
                not_started_count = stats[3] if stats[3] else 0
                rejected_other_count = stats[4] if stats[4] else 0

                # حساب إجمالي غير المجتازين (الطريقة الأولى - مباشرة من الاستعلام)
                not_passed_count = stats[5] if stats[5] else 0

                # طريقة بديلة للتأكد (جمع كل الحالات غير المجتازة)
                # not_passed_count = cancelled_count + not_started_count + rejected_other_count

                # طريقة ثالثة للتأكد (طرح المجتازين من الإجمالي)
                # not_passed_count = total_participants - passed_count

                # للتشخيص - يمكن حذف هذا السطر لاحقاً
                if total_participants > 0:
                    print(f"الدورة {course_code}: إجمالي={total_participants}, مجتاز={passed_count}, "
                          f"إلغاء={cancelled_count}, لم يباشر={not_started_count}, "
                          f"مرفوض أخرى={rejected_other_count}, غير مجتاز={not_passed_count}")

                # إضافة الصف إلى جدول العرض
                self.courses_tree.insert("", tk.END, values=(
                    course_code,  # رقم الدورة
                    course_name,  # اسم الدورة
                    course_category,  # فئة الدورة
                    start_date,  # تاريخ البداية (نظيف بدون وقت)
                    end_date,  # تاريخ النهاية (نظيف بدون وقت)
                    total_participants,  # إجمالي الملتحقين
                    passed_count,  # عدد المجتازين
                    not_passed_count  # عدد غير المجتازين
                ))

        except sqlite3.Error as e:
            # معالجة أخطاء قاعدة البيانات
            print(f"خطأ في قاعدة البيانات: {str(e)}")
            messagebox.showerror("خطأ", f"حدث خطأ في تحميل البيانات:\n{str(e)}")

        except Exception as e:
            # معالجة الأخطاء العامة - طريقة بديلة
            print(f"خطأ غير متوقع: {str(e)}")

            try:
                cursor = self.db_conn.cursor()
                cursor.execute("SELECT * FROM courses ORDER BY course_code")

                for row in cursor.fetchall():
                    course_id = row[0]
                    course_name = row[1] if len(row) > 1 else ""
                    course_code = row[2] if len(row) > 2 else ""
                    course_category = row[3] if len(row) > 3 and row[3] else ""

                    # تنظيف التواريخ
                    start_date = ""
                    if len(row) > 4 and row[4]:
                        start_date = str(row[4]).split(' ')[0]

                    end_date = ""
                    if len(row) > 5 and row[5]:
                        end_date = str(row[5]).split(' ')[0]

                    # حساب إحصائيات المشاركين - استعلام بسيط
                    cursor.execute("""
                        SELECT participant_status, COUNT(*) 
                        FROM participants 
                        WHERE course_id = ?
                        GROUP BY participant_status
                    """, (course_id,))

                    status_counts = dict(cursor.fetchall())

                    # حساب الإحصائيات من القاموس
                    total_participants = sum(status_counts.values())

                    # حساب المجتازين
                    passed_count = status_counts.get('مجتاز', 0) + status_counts.get('باشر', 0)

                    # حساب غير المجتازين (كل الباقي)
                    not_passed_count = (
                            status_counts.get('إلغاء دورة', 0) +
                            status_counts.get('لم يباشر', 0) +
                            status_counts.get('مرفوض', 0) +
                            status_counts.get('مرفوض أخرى', 0) +
                            status_counts.get('غير مجتاز', 0)
                    )

                    # إذا كان المجموع لا يتطابق، احسب الفرق
                    if (passed_count + not_passed_count) != total_participants:
                        not_passed_count = total_participants - passed_count

                    # إضافة الصف إلى الجدول
                    self.courses_tree.insert("", tk.END, values=(
                        course_code,
                        course_name,
                        course_category,
                        start_date,
                        end_date,
                        total_participants,
                        passed_count,
                        not_passed_count
                    ))

            except Exception as backup_error:
                print(f"فشلت المحاولة البديلة أيضاً: {str(backup_error)}")
                messagebox.showerror(
                    "خطأ",
                    "لم نتمكن من تحميل البيانات.\n"
                    "الرجاء التحقق من قاعدة البيانات."
                )
