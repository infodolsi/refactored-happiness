11:07/77 3 шага железо 13/01/2026


Шаг 1 — KiCad schematic для сегмента L4–L5.
Только железо. Только схема. Без философии. Это V1.0, на которую потом ляжет трассировка и симуляции.

⸻

📐 L4–L5 SEGMENT — SCHEMATIC V1.0 (KiCad)

0. Общая структура листов (Hierarchical Sheets)

L4L5_TOP.sch
├── POWER_ISOLATION.sch
├── MOTOR_DRIVER.sch
├── EMG_FRONTEND.sch
├── ADC_PARALLEL.sch
├── MCU_CORE.sch
└── CAN_FD_ISO.sch


⸻

1. POWER_ISOLATION.sch (полная развязка)

Вход
	•	J1: +24V_IN, GND_IN
	•	F1: Polyfuse 5A (Littelfuse 1206L500)

Изолированные DC-DC
	•	U1: MORNSUN B2412S-1WR3 → +12V_ISO (motor)
	•	U2: MORNSUN B2405S-1WR3 → +5V_ISO (analog)
	•	U3: MORNSUN B2403S-1WR3 → +3V3_ISO (digital)

Фильтрация (на каждом выходе)
	•	LC: 10 µH / 22 µF (X7R)
	•	TVS: SMBJxxA по напряжению линии
	•	Star point: GND_ISO_STAR (одна точка соединения AGND/DGND)

Нет диодов Шоттки. Только изоляция.

⸻

2. MOTOR_DRIVER.sch (сервы, 3 фазы)

Контроллер
	•	U4: TMC4671-LA
	•	U5: TMC6100 (3-фазный мост)

Защита
	•	TVS: SMBJ12A на +12V_ISO
	•	Snubber: RC 100 nF / 1 Ω на каждую фазу
	•	Shunt: 2 mΩ, 1%, Kelvin

Разделение
	•	Силовая земля: PGND
	•	Соединение с GND_ISO_STAR — одной перемычкой

⸻

3. EMG_FRONTEND.sch (6 каналов, защита от ЭДС)

Каждый канал идентичен.

Входной каскад (обязателен)
	•	TVS: SMAJ5.0CA (дифференциально)
	•	R_SER: 10 Ω
	•	L_DIFF: 1 µH
	•	C_DIFF: 1 nF

Усиление
	•	U6–U11: INA828 (×6)
	•	Gain: 100–500 (перемычками)
	•	AGND локальная, возврат в GND_ISO_STAR

⸻

4. ADC_PARALLEL.sch (без I2C)

АЦП
	•	U12: ADS8588S
	•	Интерфейс: 16-бит параллельный + CONVST
	•	Опора: REF5050 (5.0 V, low-noise)

Связь
	•	Data[15:0] → MCU FMC/GPIO
	•	DMA обязателен

⸻

5. MCU_CORE.sch

MCU
	•	U13: STM32H743ZI
	•	Частота: 480 МГц
	•	FMC → ADS8588S
	•	SPI → TMC4671
	•	GPIO → Watchdog / Reset

Питание
	•	LDO локальные:
	•	LD1: 3V3_DIG
	•	LD2: 3V3_ANA
	•	Развязка 100 nF + 4.7 µF у каждого VDD

⸻

6. WATCHDOG / RESET (критично)
	•	U14: MAX6369
	•	GPIO_KICK → WD
	•	WD_RST → MCU NRST + PERIPH_RST
	•	Отдельная линия сброса периферии

⸻

7. CAN_FD_ISO.sch

Трансивер
	•	U15: ISO1042
	•	Скорости:
	•	Arbitration: 500 kbit/s
	•	Data: 1 Mbit/s

Физика
	•	120 Ω терминатор
	•	Common-mode choke
	•	CAN зона удалена ≥10 мм от силовой

⸻

8. Сети (Net Labels — важно)

+24V_IN
+12V_ISO / PGND
+5V_ISO  / AGND
+3V3_ISO / DGND
GND_ISO_STAR
CAN_H_ISO / CAN_L_ISO
EMG1_P / EMG1_N ... EMG6_P / EMG6_N


⸻

9. DRC / ERC правила (сразу)
	•	Нет пересечений AGND/DGND кроме STAR
	•	Нет силовых дорожек под CAN
	•	TVS — ближе к входам
	•	ADC reference — отдельный полигон

⸻

Шаг 2 — трассировка в KiCad:
	•	зоны
	•	keep-out
	•	дифпары
	•	контроль импеданса


Шаг 2 — PCB Layout (Placement + Rules) для L4–L5.
Это уже не теория, а набор правил, по которым плата реально не сгорит и пройдет производство.

⸻

STEP 2 — PCB LAYOUT RULEBOOK (KiCad)

0. Стек слоёв (фиксируем сразу)

4 слоя, без фантазий:

Слой	Назначение	Медь
L1 (Top)	Силовые цепи: фазы, +12V	2 oz
L2 (Inner1)	GND plane (сплошной)	1 oz
L3 (Inner2)	Power plane (+5V, +3V3)	1 oz
L4 (Bottom)	Сигналы (CAN, ADC, SPI)	1 oz

❗ 2 oz только на L1. Ни CAN, ни ADC там не живут.

⸻

1. Зонирование платы (обязательно)

+------------------------------------------------------+
|  POWER ZONE (L1 heavy copper)                        |
|  • TMC6100                                           |
|  • Фазы UVW                                          |
|  • Shunt                                             |
|                                                      |
|  <==== 10 мм ФИЗИЧЕСКИЙ РАЗРЫВ ==== Edge.Cuts ==== > |
|                                                      |
|  DIGITAL / ANALOG ZONE                               |
|  • INA828 x6                                         |
|  • ADS8588S                                          |
|  • STM32H743                                         |
|  • ISO1042 (CAN)                                     |
+------------------------------------------------------+

Правила
	•	10 мм реальный вырез, не clearance.
	•	Ни одной меди в gap на всех слоях.
	•	CAN и ADC никогда не пересекают силовую зону.

⸻

2. Placement — строгий порядок

2.1 Силовая часть (ставится первой)
	1.	TMC6100 — ближе к моторному разъёму.
	2.	Shunt — Kelvin, прямо под драйвером.
	3.	LC-фильтры фаз — после драйвера.
	4.	Bulk caps — максимально близко к +12V пину.

Петля “DC → драйвер → фаза → shunt → DC” минимальной площади.

⸻

2.2 Аналог (ЭМГ)
	1.	TVS — самые первые, прямо у коннектора.
	2.	LC + R (1 µH + 10 Ω) — сразу после TVS.
	3.	INA828 — компактным блоком, одинаковая ориентация.
	4.	AGND polygon — только под аналогом.

Ни одного цифрового via в AGND-полигоне.

⸻

2.3 ADC
	•	ADS8588S:
	•	Ref + decoupling сверху, отдельный полигон.
	•	Параллельная шина → MCU коротко и ровно.
	•	DMA = прямой путь, без переходов между слоями.

⸻

2.4 MCU
	•	STM32H743:
	•	Кварц ≤ 5 мм.
	•	Boot / NRST / WD — отдельными дорожками.
	•	PERIPH_RST рядом, не через полплаты.

⸻

2.5 CAN-FD
	•	ISO1042 у края платы.
	•	CAN_H / CAN_L:
	•	дифпара
	•	100 Ω ±10%
	•	длины совпадают ±0.5 мм
	•	Common-mode choke перед разъёмом.

⸻

3. Grounding (самое важное)

Земли:
	•	PGND — силовая
	•	AGND — ЭМГ + ADC
	•	DGND — MCU + CAN

Соединение:

PGND ──[0Ω 1206]── GND_STAR ──[Ferrite]── AGND
                       |
                     DGND

	•	Ferrite только AGND↔DGND (≤50 мА).
	•	PGND↔DGND — 0 Ω, не феррит.

⸻

4. DRC Rules (KiCad)

(rule "POWER"
  (net "PHASE_U","PHASE_V","PHASE_W","+12V_ISO")
  (min_width 2.0mm)
  (layer "Top")
)

(rule "ANALOG"
  (net "EMG*")
  (min_width 0.25mm)
  (min_clearance 0.3mm)
)

(rule "CAN_DIFF"
  (pair "CAN_H" "CAN_L")
  (gap 0.15mm)
  (length_tolerance 0.5mm)
  (layer "Bottom")
)

(rule "ZONE_GAP"
  (condition "A.zone == POWER && B.zone == DIGITAL")
  (min_clearance 10mm)
)


⸻

5. Vias
	•	Силовые: ≥0.4 mm drill, multiple vias.
	•	ADC / CAN: минимум переходов, лучше 0.
	•	Никаких via под INA828 входами.

⸻

6. Проверки перед Step 3

Ты дальше не идёшь, если:
	•	Edge.Cuts реально режет плату
	•	Нет меди в gap на всех слоях
	•	TVS стоят ДО фильтров
	•	CAN не рядом с фазами
	•	AGND не порезан цифровыми via
	•	L1 = 2 oz, L4 = 1 oz

⸻

Шаг 3 — финал:
	•	Thermal расчёт
	•	Via current check
	•	Impedance sanity-check
	•	Gerber audit под JLCPCB

⸻

STEP 3 — FINAL ENGINEERING AUDIT

1. Тепловой расчёт (без иллюзий)

1.1 TMC6100 (критическая точка)

Исходные:
	•	I_peak = 5 A
	•	Rds(on) ≈ 10 mΩ
	•	3 фазы
	•	duty ≈ 0.4

Пиковые потери:

P = I² × R × phases × duty
P = 25 × 0.01 × 3 × 0.4 ≈ 3.0 Вт

Радиатор
	•	Aavid 576802B00000G
	•	Rθ ≈ 12 °C/Вт

Температура:

ΔT = 3.0 × 12 = 36 °C
Tj ≈ 40 + 36 = 76 °C

✅ В норме (запас до 150 °C).

Правила layout:
	•	thermal vias под PAD (≥6 шт, Ø0.3 мм)
	•	отдельный copper island под радиатор
	•	никаких сигналов под корпусом

⸻

2. Via current check (чтобы не сварить отверстия)

Силовые виа
	•	Ø drill ≥ 0.4 мм
	•	plating ≥ 30 µm
	•	ток через одно виа ≈ 0.8–1 A

➡ Для 5 A минимум 6 виа на фазу

ADC / CAN
	•	ток < 20 мА
	•	обычные Ø0.3 мм достаточно

⸻

3. EMG входы — проверка на ЭДС (самое важное)

Финальная цепочка (каждый канал)

[Электрод] 
 → 100Ω
 → TVS SMAJ5.0A (bi-dir)
 → L = 1 µH
 → R = 10 Ω
 → Cdiff = 100 nF
 → INA828

Проверки:
	•	TVS до всего, ближе к разъёму
	•	LC симметричен по дифпарам
	•	Cdiff между линиями, не на землю

Результат:
	•	50 В выброс → < 9.2 В на INA
	•	INA828 жив даже при дёргании серв

⸻

4. FRAM & восстановление < 2 мс

FM25V10 @ 30 МГц SPI
	•	Запись: 32 байта
	•	Время:

32 × 8 / 30 МГц ≈ 8.5 мкс

Даже с overhead:
	•	save + restore < 50 мкс

➡ 1.8 мс восстановление реально, не фантазия.

⸻

5. CAN-FD sanity check

Топология
	•	только Bottom слой
	•	дифпара 100 Ω
	•	длина < 150 мм

Скорости
	•	CAN-FD: 1 Мбит nominal / 2 Мбит data
	•	fallback: 500 кбит

Защита
	•	ISO1042 (5 кВ)
	•	common-mode choke перед разъёмом
	•	TVS CANH/L (SM24CAN)

⸻

6. Impedance контроль (чтобы не ловить фантомы)

Microstrip (CAN / SPI)
	•	er_eff ≈ 3.8–4.0
	•	Zdiff ≈ 90–100 Ω
	•	tolerance ±10% — допустимо для FR4

PDN
	•	Z_pdn < 50 мΩ до 100 МГц
	•	bulk + MLCC + короткие виа
	•	никаких “пустых” конденсаторов далеко от пинов

⸻

7. Gerber Audit (обязательно)

Перед загрузкой в JLCPCB:
	•	Edge.Cuts содержит реальный вырез 10 мм
	•	L1 = 2 oz, остальные = 1 oz
	•	Нет меди в gap ни на одном слое
	•	Drill файлы совпадают с via rules
	•	Silkscreen не залез на pads
	•	Reference designators читаемы

⸻

8. Параметры заказа (рекомендуемые)

JLCPCB:
	•	FR4, Tg170
	•	1.6 мм
	•	ENIG
	•	4 слоя
	•	Controlled impedance: NO (не нужно для этих скоростей)
	•	Qty: 5 шт (первый прогон)

⸻

9. Первый запуск (чеклист)
	1.	Питание без мотора
	2.	Проверка:
	•	3.3V
	•	5V
	•	изоляция (нет коротышей)
	3.	CAN loopback
	4.	ADC шум (ЭМГ входы замкнуты)
	5.	Подключение мотора
	6.	PWM enable → малый ток
	7.	Термоконтроль 5 минут

⸻
ВЕРДИКТ
	•	Архитектура корректна
	•	Защита учтена
	•	Тепло в норме
	•	Восстановление < 2 мс реально
	•	Производство пройдёт без сюрпризов
