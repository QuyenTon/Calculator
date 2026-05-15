# Calculator
//maytinhcamtay
#include <Wire.h>
#include <LiquidCrystal_I2C.h>
#include <Keypad.h>
#include <math.h>
#include <stdlib.h>
#include <string.h>
#include <ctype.h>


// ================= LCD CONFIG =================
LiquidCrystal_I2C lcd(0x27, 16, 2);


// ================= KEYPAD 4x4 =================
const byte ROWS = 4;
const byte COLS = 4;
char keys[ROWS][COLS] = {
  { '1', '2', '3', 'A' },
  { '4', '5', '6', 'B' },
  { '7', '8', '9', 'C' },
  { '*', '0', '#', 'D' }
};


byte rowPins[ROWS] = { 2, 3, 4, 5 };
byte colPins[COLS] = { 6, 7, 8, 9 };
Keypad keypad = Keypad(makeKeymap(keys), rowPins, colPins, ROWS, COLS);


// ================= KEYMAP (BẢNG PHÍM) =================
/*
  Phím thường:
    0-9  → chữ số
    *    → dấu chấm "."
    #    → dán Ans (kết quả vừa tính)
    C    → DEL (xóa ký tự cuối)
    D    → = (tính kết quả)
    B    → vào Mode (sin/cos/tan, PT, log)
    A    → Shift (bật/tắt)


  Shift (A) + phím:
    A+1 → +
    A+2 → -
    A+3 → *
    A+4 → /
    A+5 → .    (dấu chấm thập phân)
    A+6 → sqrt(
    A+7 → (
    A+8 → )
    A+9 → !    (giai thừa)
    A+C → ALL CLEAR (xóa hết, về màn hình chờ)
    A+0 → ^    (lũy thừa)


  Mode B:
    1 → Lượng giác (sin/cos/tan, nhập góc độ)
    2 → Giải phương trình (bậc 2/3/4)
    3 → Hàm log/exp (e^x, 10^x, log10, ln, a^b)


  Trong các sub-mode:
    0-9   → nhập chữ số
    *     → dấu chấm "."
    A + 2 → dấu âm "-" (đặt ở đầu số)
    C     → DEL (xóa ký tự cuối)
    D     → Enter (xác nhận)
*/


// ================= GLOBAL =================
const char *expr;
int pos;
uint8_t math_error = 0;
uint8_t shift_active = 0;


char input[50] = { 0 };
int idx = 0;
double last_ans = 0;


typedef enum {
  STATE_NORMAL = 0,
  STATE_MODE_SELECT,
  STATE_TRIG,
  STATE_TRIG_INPUT,
  STATE_POLY,
  STATE_POLY_INPUT,
  STATE_EXPLOG,
  STATE_EXPLOG_INPUT
} CalcState;


CalcState state = STATE_NORMAL;
uint8_t trig_func = 0;
uint8_t poly_degree = 0;
uint8_t poly_coeff_idx = 0;
double poly_coeffs[5] = { 0 };
uint8_t explog_func = 0;
uint8_t explog_arg_idx = 0;
double explog_arg1 = 0;
char sub_input[30] = { 0 };
int sub_idx = 0;


// ================= PARSER PROTOTYPES =================
double parseExpression();
double parseTerm();
double parsePower();
double parseFactorial();
double parseFactor();
double parseNumber();
// =======================================================
//  LCD HELPERS
// =======================================================


// Xóa trắng 1 dòng rồi in chuỗi bằng write() - đảm bảo + - / * hiện đúng
void lcd_print_row(uint8_t row, const char *str) {
  lcd.setCursor(0, row);
  uint8_t i = 0;
  while (str[i] && i < 16) {
    lcd.write((uint8_t)str[i]);
    i++;
  }
  while (i < 16) {
    lcd.write(' ');
    i++;
  }
}


// In dòng 0 từ buffer input (16 ký tự cuối)
void refresh_input_line() {
  int start = (idx > 16) ? (idx - 16) : 0;
  lcd.setCursor(0, 0);
  int printed = 0;
  for (int i = start; i < idx; i++) {
    lcd.write((uint8_t)input[i]);
    printed++;
  }
  while (printed < 16) {
    lcd.write(' ');
    printed++;
  }
}


// Icon "S" góc trên phải khi Shift đang bật
void update_status_icon() {
  lcd.setCursor(15, 0);
  lcd.write(shift_active ? (uint8_t)'S' : (uint8_t)' ');
}


void show_error(const char *msg) {
  lcd.clear();
  lcd_print_row(0, msg);
}


void reset_to_normal() {
  state = STATE_NORMAL;
  trig_func = 0;
  poly_degree = 0;
  poly_coeff_idx = 0;
  explog_func = 0;
  explog_arg_idx = 0;
  explog_arg1 = 0;
  shift_active = 0;
  memset(poly_coeffs, 0, sizeof(poly_coeffs));
  memset(sub_input, 0, sizeof(sub_input));
  sub_idx = 0;
}


// =======================================================
//  PARSER
// =======================================================
double factorial_val(int n) {
  if (n < 0 || n > 20) return -1;
  double r = 1;
  for (int i = 2; i <= n; i++) r *= i;
  return r;
}


int matchSqrt() {
  if (strncmp(&expr[pos], "sqrt", 4) == 0) {
    pos += 4;
    return 1;
  }
  return 0;
}


double parseNumber() {
  double result = 0;
  while (isdigit((unsigned char)expr[pos])) {
    result = result * 10 + (expr[pos] - '0');
    pos++;
  }
  if (expr[pos] == '.') {
    pos++;
    double frac = 0.1;
    while (isdigit((unsigned char)expr[pos])) {
      result += (expr[pos] - '0') * frac;
      frac *= 0.1;
      pos++;
    }
  }
  return result;
}


double parseFactor() {
  if (expr[pos] == '-') {
    pos++;
    return -parseFactor();
  }
  if (matchSqrt()) {
    if (expr[pos] == '(') {
      pos++;
      double val = parseExpression();
      if (expr[pos] == ')') pos++;
      if (val < 0) {
        math_error = 1;
        return 0;
      }
      return sqrt(val);
    }
  }
  if (expr[pos] == '(') {
    pos++;
    double result = parseExpression();
    if (expr[pos] == ')') pos++;
    return result;
  }
  return parseNumber();
}


double parseFactorial() {
  double result = parseFactor();
  while (expr[pos] == '!') {
    pos++;
    int n = (int)result;
    if (result < 0 || result != (double)n || n > 20) {
      math_error = 1;
      return 0;
    }
    result = factorial_val(n);
  }
  return result;
}


double parsePower() {
  double base = parseFactorial();
  if (expr[pos] == '^') {
    pos++;
    base = pow(base, parsePower());
  }
  return base;
}


double parseTerm() {
  double result = parsePower();
  while (expr[pos] == '*' || expr[pos] == '/') {
    char op = expr[pos++];
    double val = parsePower();
    if (op == '*') result *= val;
    else {
      if (val == 0) {
        math_error = 1;
        return 0;
      }
      result /= val;
    }
  }
  return result;
}


double parseExpression() {
  double result = parseTerm();
  while (expr[pos] == '+' || expr[pos] == '-') {
    char op = expr[pos++];
    double val = parseTerm();
    if (op == '+') result += val;
    else result -= val;
  }
  return result;
}


double parse_sub_input() {
  expr = sub_input;
  pos = 0;
  math_error = 0;
  return parseExpression();
}


// =======================================================
//  MATH SOLVERS
// =======================================================
void sort_arr(double *arr, int n) {
  for (int i = 0; i < n - 1; i++)
    for (int j = 0; j < n - i - 1; j++)
      if (arr[j] > arr[j + 1]) {
        double t = arr[j];
        arr[j] = arr[j + 1];
        arr[j + 1] = t;
      }
}


void solve_quad(double a, double b, double c) {
  if (a == 0) {
    show_error("a must != 0");
    return;
  }
  double D = b * b - 4 * a * c;
  lcd.clear();
  char buf[16];
  if (D < 0) {
    lcd_print_row(0, "No real roots");
  } else if (D == 0) {
    double x = -b / (2 * a);
    lcd.setCursor(0, 0);
    lcd.write('x');
    lcd.write('=');
    dtostrf(x, 10, 4, buf);
    lcd_print_row(0, buf);
  } else {
    double x1 = (-b + sqrt(D)) / (2 * a), x2 = (-b - sqrt(D)) / (2 * a);
    lcd.setCursor(0, 0);
    lcd.write('x');
    lcd.write('1');
    lcd.write('=');
    dtostrf(x1, 6, 3, buf);
    for (int i = 0; buf[i]; i++) lcd.write((uint8_t)buf[i]);
    lcd.setCursor(0, 1);
    lcd.write('x');
    lcd.write('2');
    lcd.write('=');
    dtostrf(x2, 6, 3, buf);
    for (int i = 0; buf[i]; i++) lcd.write((uint8_t)buf[i]);
  }
}
//Công thức cardano
void solve_cubic(double a, double b, double c, double d) {
  if (a == 0) {
    solve_quad(b, c, d);
    return;
  }
  b /= a;
  c /= a;
  d /= a;
  double p = c - b * b / 3.0;
  double q = 2 * b * b * b / 27.0 - b * c / 3.0 + d;
  double D = q * q / 4.0 + p * p * p / 27.0;
  char buf[12];
  lcd.clear();
  if (D > 0) {
    double x1 = cbrt(-q / 2.0 + sqrt(D)) + cbrt(-q / 2.0 - sqrt(D)) - b / 3.0;
    lcd.setCursor(0, 0);
    lcd.write('x');
    lcd.write('=');
    dtostrf(x1, 8, 4, buf);
    for (int i = 0; buf[i]; i++) lcd.write((uint8_t)buf[i]);
  } else if (D == 0) {
    double u = cbrt(-q / 2.0);
    double x1 = 2 * u - b / 3.0, x2 = -u - b / 3.0;
    lcd.setCursor(0, 0);
    lcd.write('x');
    lcd.write('1');
    lcd.write('=');
    dtostrf(x1, 5, 3, buf);
    for (int i = 0; buf[i]; i++) lcd.write((uint8_t)buf[i]);
    lcd.write(' ');
    lcd.write('x');
    lcd.write('2');
    lcd.write('=');
    dtostrf(x2, 5, 3, buf);
    for (int i = 0; buf[i]; i++) lcd.write((uint8_t)buf[i]);
  } else {
    double r = sqrt(-p * p * p / 27.0), theta = acos(-q / (2 * r)), m = 2.0 * cbrt(r);
    double roots[3] = { m * cos(theta / 3.0) - b / 3.0,
                        m * cos((theta + 2 * M_PI) / 3.0) - b / 3.0,
                        m * cos((theta + 4 * M_PI) / 3.0) - b / 3.0 };
    sort_arr(roots, 3);
    lcd.setCursor(0, 0);
    lcd.write('x');
    lcd.write('1');
    lcd.write('=');
    dtostrf(roots[0], 4, 2, buf);
    for (int i = 0; buf[i]; i++) lcd.write((uint8_t)buf[i]);
    lcd.write(' ');
    lcd.write('x');
    lcd.write('2');
    lcd.write('=');
    dtostrf(roots[1], 4, 2, buf);
    for (int i = 0; buf[i]; i++) lcd.write((uint8_t)buf[i]);
    lcd.setCursor(0, 1);
    lcd.write('x');
    lcd.write('3');
    lcd.write('=');
    dtostrf(roots[2], 5, 3, buf);
    for (int i = 0; buf[i]; i++) lcd.write((uint8_t)buf[i]);
  }
}


void print_roots(double *roots, int cnt) {
  char buf[10];
  lcd.setCursor(0, 0);
  if (cnt > 0) {
    lcd.write('x');
    lcd.write('1');
    lcd.write('=');
    dtostrf(roots[0], 4, 2, buf);
    for (int i = 0; buf[i]; i++) lcd.write((uint8_t)buf[i]);
  }
  if (cnt > 1) {
    lcd.write(' ');
    lcd.write('x');
    lcd.write('2');
    lcd.write('=');
    dtostrf(roots[1], 4, 2, buf);
    for (int i = 0; buf[i]; i++) lcd.write((uint8_t)buf[i]);
  }
  lcd.setCursor(0, 1);
  if (cnt > 2) {
    lcd.write('x');
    lcd.write('3');
    lcd.write('=');
    dtostrf(roots[2], 4, 2, buf);
    for (int i = 0; buf[i]; i++) lcd.write((uint8_t)buf[i]);
  }
  if (cnt > 3) {
    lcd.write(' ');
    lcd.write('x');
    lcd.write('4');
    lcd.write('=');
    dtostrf(roots[3], 4, 2, buf);
    for (int i = 0; buf[i]; i++) lcd.write((uint8_t)buf[i]);
  }
}


void solve_quartic(double a, double b, double c, double d, double e) {
  if (a == 0) {
    solve_cubic(b, c, d, e);
    return;
  }
  b /= a;
  c /= a;
  d /= a;
  e /= a;
  double p = c - 3.0 * b * b / 8.0;
  double q = b * b * b / 8.0 - b * c / 2.0 + d;
  double r = -3.0 * b * b * b * b / 256.0 + b * b * c / 16.0 - b * d / 4.0 + e;
  double sh = -b / 4.0;
  lcd.clear();


  if (fabs(q) < 1e-9) {
    double disc = p * p - 4.0 * r;
    if (disc < 0) {
      lcd_print_row(0, "No real roots");
      return;
    }
    double z1 = (-p + sqrt(disc)) / 2.0, z2 = (-p - sqrt(disc)) / 2.0;
    double roots[4];
    int cnt = 0;
    if (z1 >= 0) {
      roots[cnt++] = sqrt(z1) + sh;
      roots[cnt++] = -sqrt(z1) + sh;
    }
    if (z2 >= 0) {
      roots[cnt++] = sqrt(z2) + sh;
      roots[cnt++] = -sqrt(z2) + sh;
    }
    if (cnt == 0) {
      lcd_print_row(0, "No real roots");
      return;
    }
    sort_arr(roots, cnt);
    print_roots(roots, cnt);
  } else {
    double rc_b = p;
    double rc_c = p * p / 4.0 - r;
    double rc_d = -(q * q / 8.0);
    double pb = rc_c - rc_b * rc_b / 3.0;
    double qb = 2.0 * rc_b * rc_b * rc_b / 27.0 - rc_b * rc_c / 3.0 + rc_d;
    double Db = qb * qb / 4.0 + pb * pb * pb / 27.0;
    double m;
    if (Db >= 0) {
      m = cbrt(-qb / 2.0 + sqrt(Db)) + cbrt(-qb / 2.0 - sqrt(Db)) - rc_b / 3.0;
    } else {
      double rr = sqrt(-pb * pb * pb / 27.0);
      double th = acos((-qb / 2.0) / rr);
      m = 2.0 * cbrt(rr) * cos(th / 3.0) - rc_b / 3.0;
    }
    if (2.0 * m < 0) {
      lcd_print_row(0, "Complex roots");
      return;
    }
    double sq = sqrt(2.0 * m);
    double p1 = m + p / 2.0 - q / (2.0 * sq), p2 = m + p / 2.0 + q / (2.0 * sq);
    double roots[4];
    int cnt = 0;
    double d1 = sq * sq / 4.0 - p1, d2 = sq * sq / 4.0 - p2;
    if (d1 >= 0) {
      roots[cnt++] = sq / 2.0 + sqrt(d1) + sh;
      roots[cnt++] = sq / 2.0 - sqrt(d1) + sh;
    }
    if (d2 >= 0) {
      roots[cnt++] = -sq / 2.0 + sqrt(d2) + sh;
      roots[cnt++] = -sq / 2.0 - sqrt(d2) + sh;
    }
    if (cnt == 0) {
      lcd_print_row(0, "No real roots");
      return;
    }
    sort_arr(roots, cnt);
    print_roots(roots, cnt);
  }
}


void show_coeff_prompt(uint8_t degree, uint8_t index) {
  const char *names[] = { "a", "b", "c", "d", "e" };
  lcd.clear();
  lcd_print_row(0, "Enter ?:");
  lcd.setCursor(6, 0);
  lcd.write((uint8_t)names[index][0]);
  lcd.setCursor(0, 1);
  lcd_print_row(1, "");
}


// =======================================================
//  SUB-INPUT HANDLER
// =======================================================
bool process_sub_input(char key) {
  if (shift_active) {
    shift_active = 0;
    update_status_icon();
    if (key == '2' && sub_idx == 0) {
      sub_input[sub_idx++] = '-';
      sub_input[sub_idx] = '\0';
      lcd.setCursor(0, 1);
      lcd_print_row(1, sub_input);
    }
    return false;
  }


  if (key == 'D') return true;  // Enter


  if (key == 'C') {  // DEL
    if (sub_idx > 0) {
      sub_input[--sub_idx] = '\0';
      lcd_print_row(1, sub_input);
    }
    return false;
  }


  if (key == 'A') {  // Shift toggle
    shift_active = !shift_active;
    update_status_icon();
    return false;
  }


  if (key == '*') {  // Dấu chấm
    if (sub_idx < (int)sizeof(sub_input) - 1) {
      sub_input[sub_idx++] = '.';
      sub_input[sub_idx] = '\0';
      lcd_print_row(1, sub_input);
    }
    return false;
  }


  if (isdigit(key)) {
    if (sub_idx < (int)sizeof(sub_input) - 1) {
      sub_input[sub_idx++] = key;
      sub_input[sub_idx] = '\0';
      lcd_print_row(1, sub_input);
    }
    return false;
  }


  return false;
}


// =======================================================
//  APPEND TO MAIN INPUT
// =======================================================
void append_input(const char *str) {
  int len = strlen(str);
  if (idx + len >= (int)sizeof(input) - 1) return;
  strcat(input, str);
  idx += len;
  refresh_input_line();
  update_status_icon();
}


// =======================================================
//  SETUP ĐÃ ĐƯỢC CHỈNH SỬA CHO PHẦN CỨNG
// =======================================================
void setup() {
  // Thêm delay để mạch I2C và LCD có thời gian ổn định nguồn
  delay(250);


  // Khởi tạo bus I2C
  Wire.begin();


  lcd.init();
  lcd.backlight();
  lcd.clear();
  lcd_print_row(0, "Arduino Calc");
  lcd_print_row(1, "D=Calc  C=DEL");
  delay(1200);
  lcd.clear();
  lcd_print_row(0, "");
  lcd_print_row(1, "A=Shift  B=Mode");
}


// =======================================================
//  MAIN LOOP
// =======================================================
void loop() {
  char key = keypad.getKey(); -> Đọc phím
  if (!key) return; //không có phím -> bỏ qua
//Phím A = shift
  if (key == 'A' && state == STATE_NORMAL) {
    shift_active = !shift_active;
    update_status_icon(); //hiện “s” ở góc lcd
    return;
  }
// A + C = clear all
  if (shift_active && key == 'C') {
    shift_active = 0;
    idx = 0;
    memset(input, 0, sizeof(input));
    math_error = 0;
    reset_to_normal();
    lcd.clear();
    lcd_print_row(0, "");
    lcd_print_row(1, "A=Shift  B=Mode");
    return;
  }


  if (state == STATE_NORMAL) {
    if (shift_active) {
      shift_active = 0;
      update_status_icon();
      switch (key) {
        case '1': append_input("+"); break;
        case '2': append_input("-"); break;
        case '3': append_input("*"); break;
        case '4': append_input("/"); break;
        case '5': append_input("."); break;
        case '6': append_input("sqrt("); break;
        case '7': append_input("("); break;
        case '8': append_input(")"); break;
        case '9': append_input("!"); break;
        case '0': append_input("^"); break;
      }
      return;
    }


    switch (key) {
      case 'B':
        state = STATE_MODE_SELECT;
        lcd.clear();
        lcd_print_row(0, "1=Tri 2=Eq");
        lcd_print_row(1, "3=Log/Exp");
        break;


      case 'C':
        if (idx > 0) {
          input[--idx] = '\0';
          refresh_input_line();
        }
        break;


      case 'D':
        input[idx] = '\0';
        expr = input;
        pos = 0;
        math_error = 0;
        {
          double result = parseExpression();
          lcd.clear();
          if (math_error) {
            lcd_print_row(0, "Math Error");
          } else {
            last_ans = result;
            char buf[20];
            dtostrf(result, 12, 6, buf);
            char *p = buf;
            while (*p == ' ') p++;
            lcd.setCursor(0, 0);
            while (*p) lcd.write((uint8_t)*p++);
          }
          lcd_print_row(1, "D=cont  B=mode");
        }
        idx = 0;
        memset(input, 0, sizeof(input));
        break;


      case '*':
        append_input(".");
        break;


      case '#':
        {
          char ans_buf[20];
          dtostrf(last_ans, 0, 6, ans_buf);
          char *p = ans_buf;
          while (*p == ' ') p++;
          append_input(p);
        }
        break;


      default:
        if (isdigit(key)) {
          char s[2] = { key, '\0' };
          append_input(s);
        }
        break;
    }
    return;
  }


  if (state == STATE_MODE_SELECT) {
    if (key == 'D' || key == 'C') {
      reset_to_normal();
      lcd.clear();
      refresh_input_line();
      return;
    }
    if (key == '1') {
      state = STATE_TRIG;
      lcd.clear();
      lcd_print_row(0, "1=sin 2=cos");
      lcd_print_row(1, "3=tan");
    } else if (key == '2') {
      state = STATE_POLY;
      lcd.clear();
      lcd_print_row(0, "Degree?");
      lcd_print_row(1, "2 / 3 / 4");
    } else if (key == '3') {
      state = STATE_EXPLOG;
      lcd.clear();
      lcd_print_row(0, "1=e^x 2=10^x");
      lcd_print_row(1, "3=log 4=ln 5=a^b");
    }
    return;
  }


  if (state == STATE_TRIG) {
    if (key == 'D' || key == 'C') {
      reset_to_normal();
      lcd.clear();
      refresh_input_line();
      return;
    }
    if (key >= '1' && key <= '3') {
      trig_func = key - '0';
      state = STATE_TRIG_INPUT;
      memset(sub_input, 0, sizeof(sub_input));
      sub_idx = 0;
      lcd.clear();
      if (trig_func == 1) lcd_print_row(0, "sin - angle(deg):");
      else if (trig_func == 2) lcd_print_row(0, "cos - angle(deg):");
      else lcd_print_row(0, "tan - angle(deg):");
      lcd_print_row(1, "");
    }
    return;
  }


  if (state == STATE_TRIG_INPUT) {
    if (process_sub_input(key)) {
      sub_input[sub_idx] = '\0';
      double deg = parse_sub_input();
      double rad = deg * M_PI / 180.0;
      double res = 0;
      uint8_t err = 0;
      if (trig_func == 1) res = sin(rad);
      else if (trig_func == 2) res = cos(rad);
      else {
        if (fabs(cos(rad)) < 1e-9) err = 1;
        else res = tan(rad);
      }
      lcd.clear();
      if (math_error || err) {
        lcd_print_row(0, "Undefined");
      } else {
        last_ans = res;
        char buf[20];
        dtostrf(res, 12, 6, buf);
        char *p = buf;
        while (*p == ' ') p++;
        lcd.setCursor(0, 0);
        while (*p) lcd.write((uint8_t)*p++);
      }
      lcd_print_row(1, "D=cont  B=mode");
      reset_to_normal();
    }
    return;
  }


  if (state == STATE_POLY) {
    if (key == 'D' || key == 'C') {
      reset_to_normal();
      lcd.clear();
      refresh_input_line();
      return;
    }
    if (key == '2' || key == '3' || key == '4') {
      poly_degree = key - '0';
      poly_coeff_idx = 0;
      memset(poly_coeffs, 0, sizeof(poly_coeffs));
      memset(sub_input, 0, sizeof(sub_input));
      sub_idx = 0;
      state = STATE_POLY_INPUT;
      show_coeff_prompt(poly_degree, 0);
    }
    return;
  }


  if (state == STATE_POLY_INPUT) {
    if (process_sub_input(key)) {
      sub_input[sub_idx] = '\0';
      poly_coeffs[poly_coeff_idx] = parse_sub_input();
      poly_coeff_idx++;
      uint8_t needed = poly_degree + 1;
      if (poly_coeff_idx >= needed) {
        if (poly_degree == 2) solve_quad(poly_coeffs[0], poly_coeffs[1], poly_coeffs[2]);
        else if (poly_degree == 3) solve_cubic(poly_coeffs[0], poly_coeffs[1], poly_coeffs[2], poly_coeffs[3]);
        else solve_quartic(poly_coeffs[0], poly_coeffs[1], poly_coeffs[2], poly_coeffs[3], poly_coeffs[4]);
        lcd_print_row(1, "D=cont  B=mode");
        reset_to_normal();
      } else {
        memset(sub_input, 0, sizeof(sub_input));
        sub_idx = 0;
        show_coeff_prompt(poly_degree, poly_coeff_idx);
      }
    }
    return;
  }


  if (state == STATE_EXPLOG) {
    if (key == 'D' || key == 'C') {
      reset_to_normal();
      lcd.clear();
      refresh_input_line();
      return;
    }
    if (key >= '1' && key <= '5') {
      explog_func = key - '0';
      explog_arg_idx = 0;
      explog_arg1 = 0;
      memset(sub_input, 0, sizeof(sub_input));
      sub_idx = 0;
      state = STATE_EXPLOG_INPUT;
      lcd.clear();
      switch (explog_func) {
        case 1: lcd_print_row(0, "e^x  enter x:"); break;
        case 2: lcd_print_row(0, "10^x enter x:"); break;
        case 3: lcd_print_row(0, "log10(x):"); break;
        case 4: lcd_print_row(0, "ln(x):"); break;
        case 5: lcd_print_row(0, "a^b  enter a:"); break;
      }
      lcd_print_row(1, "");
    }
    return;
  }


  if (state == STATE_EXPLOG_INPUT) {
    if (process_sub_input(key)) {
      sub_input[sub_idx] = '\0';
      double val = parse_sub_input();


      if (explog_func == 5 && explog_arg_idx == 0) {
        explog_arg1 = val;
        explog_arg_idx = 1;
        memset(sub_input, 0, sizeof(sub_input));
        sub_idx = 0;
        lcd.clear();
        lcd_print_row(0, "a^b  enter b:");
        lcd_print_row(1, "");
        return;
      }


      double res = 0;
      uint8_t err = 0;
      switch (explog_func) {
        case 1: res = exp(val); break;
        case 2: res = pow(10.0, val); break;
        case 3:
          if (val <= 0) err = 1;
          else res = log10(val);
          break;
        case 4:
          if (val <= 0) err = 1;
          else res = log(val);
          break;
        case 5: res = pow(explog_arg1, val); break;
      }
      lcd.clear();
      if (math_error || err) {
        lcd_print_row(0, "Math Error");
      } else {
        last_ans = res;
        char buf[20];
        dtostrf(res, 12, 6, buf);
        char *p = buf;
        while (*p == ' ') p++;
        lcd.setCursor(0, 0);
        while (*p) lcd.write((uint8_t)*p++);
      }
      lcd_print_row(1, "D=cont  B=mode");
      reset_to_normal();
    }
    return;
  }
}

