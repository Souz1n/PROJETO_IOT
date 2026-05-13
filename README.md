# 🔒 Cofre Eletrônico com Arduino

Projeto IoT desenvolvido para a disciplina de Ciência da Computação — Centro Universitário FEI.

**Integrantes:**
- Brayan André da Costa — RA: 22124005-4
- Gustavo Souza Alvarenga — RA: 22124058-3

---

## 📋 Sobre o Projeto

Um cofre eletrônico controlado por senha, desenvolvido com Arduino UNO. O sistema permite abrir e fechar o cofre via senha digitada em um teclado 4x4, exibe mensagens em um display LCD 16x2 e possui múltiplas funcionalidades de segurança como alarme de luz, bloqueio por tentativas incorretas e botão de emergência.

---

## ⚙️ Funcionalidades

- ✅ Controle de acesso por senha numérica (até 8 dígitos)
- ✅ Abertura/fechamento físico via servo motor
- ✅ Display LCD com mensagens de status em tempo real
- ✅ Alarme sonoro ao detectar luz com o cofre **fechado** (tentativa de arrombamento)
- ✅ Aviso sonoro ao detectar luz com o cofre **aberto**
- ✅ Bloqueio de 10 segundos após 3 tentativas incorretas de senha
- ✅ Botão de emergência com interrupção por hardware (fecha o cofre imediatamente)
- ✅ Contagem regressiva no display durante o bloqueio

---

## 🧰 Componentes Utilizados

| Componente | Qtd | Pino Arduino |
|---|---|---|
| Arduino UNO R3 | 1 | — |
| Display LCD 16x2 | 1 | D8 a D13 |
| Teclado Matricial 4x4 | 1 | D4–D7 / A2–A5 |
| Servo Motor | 1 | D3 |
| LDR (Fotoresistor) | 1 | A0 |
| Resistor 10kΩ | 2 | (divisor do LDR) |
| Resistor 220kΩ | 1 | (LCD) |
| Buzzer | 1 | A1 |
| Botão | 1 | D2 (INT0) |
| Potenciômetro | 1 | Contraste LCD |
| Protoboard + Jumpers | — | — |

---

## 🔌 Diagrama de Conexões

```
Arduino        Componente
─────────────────────────────────────
D13       →    LCD RS
D12       →    LCD EN
D11       →    LCD D4
D10       →    LCD D5
D9        →    LCD D6
D8        →    LCD D7
D7        →    Teclado Linha 1
D6        →    Teclado Linha 2
D5        →    Teclado Linha 3
D4        →    Teclado Linha 4
D3        →    Servo (sinal)
D2        →    Botão Emergência (INT0)
A5        →    Teclado Coluna 4
A4        →    Teclado Coluna 3
A3        →    Teclado Coluna 2
A2        →    Teclado Coluna 1
A1        →    Buzzer
A0        →    LDR (com resistor 10kΩ para GND)
5V        →    VCC (LCD, Teclado, Servo, LDR)
GND       →    GND (todos os componentes)
```

### Ligação do LDR
```
5V ── [LDR] ── ponto de junção ── A0
                     │
                  [10kΩ]
                     │
                    GND
```

---

## 🧠 Como Funciona (Máquina de Estados)

O sistema opera com **5 estados**:

| Estado | Descrição |
|---|---|
| **TRANCADO** | Estado inicial. Aguarda `#` para iniciar. Monitora luz (arrombamento). |
| **DIGITANDO** | Usuário digita a senha. `*` cancela, `#` confirma. |
| **ABERTO** | Servo a 90°. Monitora luz. `#` fecha o cofre. |
| **BLOQUEADO** | Bloqueio de 10s com contagem no display. |
| **EMERGÊNCIA** | Botão D2 pressionado: fecha tudo e bloqueia. |

---

## 💻 Código

```cpp
#include <LiquidCrystal.h>
#include <Keypad.h>
#include <Servo.h>

// --- LCD: pinos RS, EN, D4, D5, D6, D7 ---
LiquidCrystal lcd(13, 12, 11, 10, 9, 8);

// --- Teclado 4x4 ---
const byte ROWS = 4, COLS = 4;
char keys[ROWS][COLS] = {
  {'1','2','3','A'},
  {'4','5','6','B'},
  {'7','8','9','C'},
  {'*','0','#','D'}
};
byte rowPins[ROWS] = {7, 6, 5, 4};
byte colPins[COLS] = {A5, A4, A3, A2};
Keypad teclado = Keypad(makeKeymap(keys), rowPins, colPins, ROWS, COLS);

// --- Servo ---
Servo servo;
#define PINO_SERVO  3

// --- Sensores e atuadores ---
#define PINO_LDR     A0
#define PINO_BUZZER  A1
#define PINO_EMG     2    // botão de emergência (INT0)

// --- Configuracoes ---
#define SENHA       "2026"
#define MAX_ERROS   3
#define T_BLOQUEIO  10000
#define LIMITE_LDR  600

// --- Estados: 0=trancado, 1=digitando, 2=aberto, 3=bloqueado, 4=emergencia ---
int estado = 0;

String senha = "";
int erros = 0;
unsigned long tempoBloq = 0;

volatile bool emg = false;

// Interrupção por hardware do botão de emergência
void ISR_emg() {
  emg = true;
}

void setup() {
  pinMode(PINO_BUZZER, OUTPUT);
  pinMode(PINO_EMG, INPUT_PULLUP);

  // INT0 no pino D2 — detecta borda de descida (botão pressionado)
  attachInterrupt(digitalPinToInterrupt(PINO_EMG), ISR_emg, FALLING);

  servo.attach(PINO_SERVO);
  servo.write(0); // inicia fechado

  lcd.begin(16, 2);
  mostrarLCD("Cofre Fechado", "# p/ abrir");
}

void loop() {
  // Verifica emergência antes de qualquer estado
  if (emg) {
    emg = false;
    estado = 4;
  }

  if      (estado == 0) estadoTrancado();
  else if (estado == 1) estadoDigitando();
  else if (estado == 2) estadoAberto();
  else if (estado == 3) estadoBloqueado();
  else if (estado == 4) estadoEmergencia();
}

// ── ESTADO 0: TRANCADO ──────────────────────────────────────────
void estadoTrancado() {
  char tecla = teclado.getKey();

  // Detecta luz mesmo com cofre fechado (tentativa de arrombamento)
  int ldr = analogRead(PINO_LDR);
  if (ldr > LIMITE_LDR) {
    mostrarLCD("ALERTA!", "Tentativa arromb.");
    alarme();
    mostrarLCD("Cofre Fechado", "# p/ abrir");
  }

  // Pressionar # inicia a digitação da senha
  if (tecla == '#') {
    senha = "";
    estado = 1;
    mostrarLCD("Digite a senha:", "");
  }
}

// ── ESTADO 1: DIGITANDO ─────────────────────────────────────────
void estadoDigitando() {
  char tecla = teclado.getKey();
  if (!tecla) return;

  if (tecla == '*') {         // * cancela e volta ao trancado
    senha = "";
    estado = 0;
    mostrarLCD("Cofre Fechado", "# p/ abrir");
    return;
  }

  if (tecla == '#') {         // # confirma a senha
    checarSenha();
    return;
  }

  // Adiciona dígito à senha (máx 8 caracteres), exibe como *
  if (senha.length() < 8) {
    senha += tecla;
    lcd.setCursor(0, 1);
    for (int i = 0; i < (int)senha.length(); i++) lcd.print("*");
  }
}

// ── VALIDAÇÃO DA SENHA ──────────────────────────────────────────
void checarSenha() {
  if (senha == SENHA) {
    // Senha correta: abre o cofre
    erros = 0;
    estado = 2;
    servo.write(90);    // gira servo para abrir
    bipSucesso();
    mostrarLCD("ACESSO LIBERADO!", "COFRE ABERTO");

  } else {
    // Senha errada: incrementa erros
    erros++;
    bipErro();

    if (erros >= MAX_ERROS) {
      // Atingiu limite: bloqueia o sistema
      estado = 3;
      tempoBloq = millis();
      mostrarLCD("ACESSO NEGADO!", "Aguarde 10s...");
      alarme();

    } else {
      // Ainda tem tentativas restantes
      String msg = "Erros: ";
      msg += erros;
      msg += "/";
      msg += MAX_ERROS;
      mostrarLCD("Senha errada!", msg);
      delay(2000);
      senha = "";
      estado = 1;
      mostrarLCD("Digite a senha:", "");
    }
  }
}

// ── ESTADO 2: ABERTO ────────────────────────────────────────────
void estadoAberto() {
  char tecla = teclado.getKey();
  int ldr = analogRead(PINO_LDR);

  // Aviso sonoro se detectar luz com cofre aberto
  if (ldr > LIMITE_LDR) {
    mostrarLCD("ALARME DE LUZ!", "Verifique cofre");
    digitalWrite(PINO_BUZZER, HIGH);
    delay(200);
    digitalWrite(PINO_BUZZER, LOW);
    delay(500);
    mostrarLCD("Cofre Aberto", "# p/ fechar");
  }

  // # fecha o cofre
  if (tecla == '#') {
    servo.write(0);
    estado = 0;
    mostrarLCD("Cofre Fechado", "# p/ abrir");
  }
}

// ── ESTADO 3: BLOQUEADO ─────────────────────────────────────────
void estadoBloqueado() {
  unsigned long passado = millis() - tempoBloq;

  if (passado >= T_BLOQUEIO) {
    // Tempo de bloqueio encerrado
    erros = 0;
    estado = 0;
    mostrarLCD("Cofre Fechado", "# p/ abrir");

  } else {
    // Exibe contagem regressiva
    int seg = (T_BLOQUEIO - passado) / 1000;
    lcd.setCursor(0, 1);
    lcd.print("Aguarde: ");
    lcd.print(seg);
    lcd.print("s   ");
    delay(300);
  }
}

// ── ESTADO 4: EMERGÊNCIA ────────────────────────────────────────
void estadoEmergencia() {
  servo.write(0);   // fecha o cofre imediatamente
  alarme();
  estado = 3;
  tempoBloq = millis();
  mostrarLCD("EMERGENCIA!", "Bloqueado 10s");
}

// ── UTILITÁRIOS ─────────────────────────────────────────────────

void mostrarLCD(String l1, String l2) {
  lcd.clear();
  lcd.setCursor(0, 0); lcd.print(l1);
  lcd.setCursor(0, 1); lcd.print(l2);
}

// Dois bips curtos — acesso liberado
void bipSucesso() {
  digitalWrite(PINO_BUZZER, HIGH); delay(100);
  digitalWrite(PINO_BUZZER, LOW);  delay(100);
  digitalWrite(PINO_BUZZER, HIGH); delay(100);
  digitalWrite(PINO_BUZZER, LOW);
}

// Um bip longo — senha errada
void bipErro() {
  digitalWrite(PINO_BUZZER, HIGH); delay(600);
  digitalWrite(PINO_BUZZER, LOW);
}

// Cinco bips rápidos — alarme/emergência
void alarme() {
  for (int i = 0; i < 5; i++) {
    digitalWrite(PINO_BUZZER, HIGH); delay(200);
    digitalWrite(PINO_BUZZER, LOW);  delay(150);
  }
}
```

---

## 📦 Bibliotecas Necessárias

Instale pela **IDE Arduino** em `Sketch > Incluir Biblioteca > Gerenciar Bibliotecas`:

| Biblioteca | Como instalar |
|---|---|
| `LiquidCrystal` | Já incluída na IDE Arduino |
| `Keypad` | Buscar por **"Keypad" by Mark Stanley** |
| `Servo` | Já incluída na IDE Arduino |

---

## 🚀 Como Usar

1. Monte o circuito conforme o diagrama de conexões
2. Instale as bibliotecas necessárias na IDE Arduino
3. Carregue o código no Arduino UNO
4. A senha padrão é **`2026`**
5. Pressione `#` no teclado para iniciar a digitação
6. Digite a senha e confirme com `#`
7. Para fechar o cofre, pressione `#` novamente

---

## 🔐 Senhas e Configurações

Para alterar as configurações, edite as linhas no início do código:

```cpp
#define SENHA      "2026"   // altere para sua senha
#define MAX_ERROS  3        // tentativas antes do bloqueio
#define T_BLOQUEIO 10000    // tempo de bloqueio em milissegundos
#define LIMITE_LDR 600      // sensibilidade do sensor de luz (0–1023)
```
