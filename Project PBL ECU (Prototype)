import sys
import json
import random
import serial
from serial.tools import list_ports

from PyQt5.QtWidgets import (
    QApplication, QMainWindow, QWidget, QLabel, 
    QGridLayout, QVBoxLayout, QHBoxLayout, QFrame, QPushButton
)
from PyQt5.QtCore import QTimer, Qt
from PyQt5.QtGui import QFont
import pyqtgraph as pg

class EVDashboard(QMainWindow):
    def __init__(self):
        super().__init__()
        self.setWindowTitle("STM32-Based EV Multi-Sensor Real-Time Dashboard")
        self.resize(1280, 720)
        self.setStyleSheet("background-color: #0f172a; color: #f8fafc;")

        # Inisialisasi Serial
        self.ser = None
        self.init_serial()

        # Buffer Data Grafik
        self.max_points = 50
        self.voltage_history = []
        self.current_history = []
        self.speed_history = []

        # Setup UI
        self.init_ui()

        # Timer Refresh Data (100 ms)
        self.timer = QTimer()
        self.timer.setInterval(100)
        self.timer.timeout.connect(self.update_telemetry)
        self.timer.start()

    def init_serial(self):
        """Mencari port COM STM32 secara otomatis."""
        ports = list(list_ports.comports())
        for p in ports:
            if "STM" in p.description or "CH340" in p.description or "CP210" in p.description:
                try:
                    self.ser = serial.Serial(p.device, 115200, timeout=0.1)
                    print(f"[+] Terhubung ke STM32 pada port {p.device}")
                    break
                except Exception as e:
                    print(f"[-] Gagal membuka port {p.device}: {e}")

    def create_card(self, title, default_val, unit, color):
        """Helper untuk membuat kartu indikator angka."""
        card = QFrame()
        card.setStyleSheet(f"""
            QFrame {{
                background-color: #1e293b;
                border: 1px solid #334155;
                border-radius: 10px;
                padding: 10px;
            }}
        """)
        layout = QVBoxLayout(card)
        
        lbl_title = QLabel(title)
        lbl_title.setStyleSheet("color: #94a3b8; font-size: 12px; font-weight: bold;")
        
        lbl_val = QLabel(f"{default_val} {unit}")
        lbl_val.setStyleSheet(f"color: {color}; font-size: 28px; font-weight: bold;")
        lbl_val.setAlignment(Qt.AlignLeft | Qt.AlignVCenter)

        layout.addWidget(lbl_title)
        layout.addWidget(lbl_val)
        return card, lbl_val

    def init_ui(self):
        main_widget = QWidget()
        self.setCentralWidget(main_widget)
        main_layout = QVBoxLayout(main_widget)

        # 1. Header
        header_layout = QHBoxLayout()
        title_label = QLabel("EV TELEMETRY MONITORING SYSTEM (STM32)")
        title_label.setFont(QFont("Arial", 16, QFont.Bold))
        title_label.setStyleSheet("color: #38bdf8;")

        self.status_label = QLabel("STATUS: SIMULATOR MODE" if not self.ser else "STATUS: CONNECTED (STM32)")
        status_color = "#f59e0b" if not self.ser else "#10b981"
        self.status_label.setStyleSheet(f"color: {status_color}; font-size: 12px; font-weight: bold;")

        header_layout.addWidget(title_label)
        header_layout.addStretch()
        header_layout.addWidget(self.status_label)
        main_layout.addLayout(header_layout)

        # 2. Grid Kartu Sensor Utama (VSS, Baterai, Suhu, O2)
        grid_cards = QGridLayout()

        self.card_vss, self.lbl_vss = self.create_card("1. KECEPATAN (VSS)", "0", "km/h", "#38bdf8")
        self.card_volt, self.lbl_volt = self.create_card("2A. TEGANGAN BATERAI", "0.0", "V", "#10b981")
        self.card_curr, self.lbl_curr = self.create_card("2B. ARUS BATERAI", "0.0", "A", "#f59e0b")
        self.card_temp, self.lbl_temp = self.create_card("3. SUHU AMBIENT", "0.0", "°C", "#f43f5e")
        self.card_o2, self.lbl_o2 = self.create_card("6. SENSOR OKSIGEN (λ)", "1.00", "Lambda", "#a855f7")

        grid_cards.addWidget(self.card_vss, 0, 0)
        grid_cards.addWidget(self.card_volt, 0, 1)
        grid_cards.addWidget(self.card_curr, 0, 2)
        grid_cards.addWidget(self.card_temp, 0, 3)
        grid_cards.addWidget(self.card_o2, 0, 4)

        main_layout.addLayout(grid_cards)

        # 3. Middle Section: Status TPMS & Keamanan (Pintu / Sabuk)
        middle_layout = QHBoxLayout()

        # 4. Sensor Tekanan Ban (TPMS)
        tpms_box = QFrame()
        tpms_box.setStyleSheet("background-color: #1e293b; border: 1px solid #334155; border-radius: 10px; padding: 10px;")
        tpms_layout = QGridLayout(tpms_box)
        
        lbl_tpms_title = QLabel("4. TEKANAN BAN (TPMS)")
        lbl_tpms_title.setStyleSheet("color: #94a3b8; font-size: 12px; font-weight: bold;")
        tpms_layout.addWidget(lbl_tpms_title, 0, 0, 1, 2)

        self.lbl_tpms_fl = QLabel("Depan Kiri: 33 PSI")
        self.lbl_tpms_fr = QLabel("Depan Kanan: 33 PSI")
        self.lbl_tpms_rl = QLabel("Belakang Kiri: 32 PSI")
        self.lbl_tpms_rr = QLabel("Belakang Kanan: 32 PSI")

        for i, lbl in enumerate([self.lbl_tpms_fl, self.lbl_tpms_fr, self.lbl_tpms_rl, self.lbl_tpms_rr]):
            lbl.setStyleSheet("color: #e2e8f0; font-size: 13px; font-weight: bold; background-color: #0f172a; padding: 8px; border-radius: 5px;")
            tpms_layout.addWidget(lbl, (i//2)+1, i%2)

        middle_layout.addWidget(tpms_box, stretch=1)

        # 5. Sensor Pintu & Sabuk Pengaman
        safety_box = QFrame()
        safety_box.setStyleSheet("background-color: #1e293b; border: 1px solid #334155; border-radius: 10px; padding: 10px;")
        safety_layout = QVBoxLayout(safety_box)

        lbl_safety_title = QLabel("5. STATUS KEAMANAN INTERIOR")
        lbl_safety_title.setStyleSheet("color: #94a3b8; font-size: 12px; font-weight: bold;")
        safety_layout.addWidget(lbl_safety_title)

        self.lbl_doors = QLabel("Pintu: SEMUA TERTUTUP")
        self.lbl_doors.setStyleSheet("color: #10b981; font-size: 13px; font-weight: bold; background-color: #0f172a; padding: 8px; border-radius: 5px;")
        
        self.lbl_belts = QLabel("Sabuk Pengaman: TERPASANG")
        self.lbl_belts.setStyleSheet("color: #10b981; font-size: 13px; font-weight: bold; background-color: #0f172a; padding: 8px; border-radius: 5px;")

        safety_layout.addWidget(self.lbl_doors)
        safety_layout.addWidget(self.lbl_belts)

        middle_layout.addWidget(safety_box, stretch=1)
        main_layout.addLayout(middle_layout)

        # 6. Grafik Real-time (PyQtGraph)
        pg.setConfigOption('background', '#1e293b')
        pg.setConfigOption('foreground', '#94a3b8')

        self.graph_widget = pg.PlotWidget()
        self.graph_widget.setTitle("Grafik Telemetri Baterai (V & I)", color="#f8fafc", size="11pt")
        self.graph_widget.showGrid(x=True, y=True, alpha=0.3)
        self.graph_widget.addLegend()

        self.curve_volt = self.graph_widget.plot(name="Tegangan (V)", pen=pg.mkPen(color="#10b981", width=2))
        self.curve_curr = self.graph_widget.plot(name="Arus (A)", pen=pg.mkPen(color="#f59e0b", width=2))

        main_layout.addWidget(self.graph_widget, stretch=2)

    def update_telemetry(self):
        """Membaca data serial atau menjalankan simulasi jika serial terputus."""
        data = None

        if self.ser and self.ser.in_waiting:
            try:
                line = self.ser.readline().decode('utf-8').strip()
                data = json.loads(line)
            except Exception:
                data = None

        # Jika STM32 tidak tersambung, gunakan data simulasi acak
        if not data:
            data = {
                "speed": round(random.uniform(40.0, 90.0), 1),
                "voltage": round(random.uniform(72.0, 78.0), 1),
                "current": round(random.uniform(10.0, 40.0), 1),
                "temp": round(random.uniform(28.0, 36.0), 1),
                "tpms": [random.choice([32, 33, 34]) for _ in range(4)],
                "door_open": random.choice([False, False, False, True]),
                "belt_fastened": random.choice([True, True, True, False]),
                "o2_lambda": round(random.uniform(0.98, 1.03), 2)
            }

        # Update Tampilan Kartu
        self.lbl_vss.setText(f"{data.get('speed', 0)} km/h")
        self.lbl_volt.setText(f"{data.get('voltage', 0.0)} V")
        self.lbl_curr.setText(f"{data.get('current', 0.0)} A")
        self.lbl_temp.setText(f"{data.get('temp', 0.0)} °C")
        self.lbl_o2.setText(f"{data.get('o2_lambda', 1.0)} λ")

        # Update TPMS
        tpms = data.get('tpms', [33, 33, 32, 32])
        self.lbl_tpms_fl.setText(f"Depan Kiri: {tpms[0]} PSI")
        self.lbl_tpms_fr.setText(f"Depan Kanan: {tpms[1]} PSI")
        self.lbl_tpms_rl.setText(f"Belakang Kiri: {tpms[2]} PSI")
        self.lbl_tpms_rr.setText(f"Belakang Kanan: {tpms[3]} PSI")

        # Update Status Keamanan
        door = data.get('door_open', False)
        belt = data.get('belt_fastened', True)

        if door:
            self.lbl_doors.setText("Pintu: TERBUKA (WARNING!)")
            self.lbl_doors.setStyleSheet("color: #f43f5e; font-size: 13px; font-weight: bold; background-color: #0f172a; padding: 8px; border-radius: 5px;")
        else:
            self.lbl_doors.setText("Pintu: SEMUA TERTUTUP")
            self.lbl_doors.setStyleSheet("color: #10b981; font-size: 13px; font-weight: bold; background-color: #0f172a; padding: 8px; border-radius: 5px;")

        if not belt:
            self.lbl_belts.setText("Sabuk: BELUM TERPASANG!")
            self.lbl_belts.setStyleSheet("color: #f43f5e; font-size: 13px; font-weight: bold; background-color: #0f172a; padding: 8px; border-radius: 5px;")
        else:
            self.lbl_belts.setText("Sabuk: TERPASANG")
            self.lbl_belts.setStyleSheet("color: #10b981; font-size: 13px; font-weight: bold; background-color: #0f172a; padding: 8px; border-radius: 5px;")

        # Update Data Grafik
        self.voltage_history.append(data.get('voltage', 0.0))
        self.current_history.append(data.get('current', 0.0))

        if len(self.voltage_history) > self.max_points:
            self.voltage_history.pop(0)
            self.current_history.pop(0)

        self.curve_volt.setData(self.voltage_history)
        self.curve_curr.setData(self.current_history)

if __name__ == "__main__":
    app = QApplication(sys.argv)
    window = EVDashboard()
    window.show()
    sys.exit(app.exec_())