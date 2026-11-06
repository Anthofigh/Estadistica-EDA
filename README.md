#include <iostream>
#include <vector>
#include <algorithm>
#include <cmath>
#include <iomanip>
#include <sstream>
#include <string>
#ifdef _WIN32
#include <windows.h>
#endif
using namespace std;

// ---------- Funciones auxiliares ----------
string rep(const string& s, int n) {            // repite un texto n veces
    string r;
    for (int i = 0; i < n; i++) r += s;
    return r;
}

int largoUtf8(const string& s) {                // cuenta caracteres visibles
    int c = 0;
    for (unsigned char ch : s)
        if ((ch & 0xC0) != 0x80) c++;
    return c;
}

string izq(const string& s, int ancho) {        // alinea a la izquierda
    return s + string(max(0, ancho - largoUtf8(s)), ' ');
}

string der(const string& s, int ancho) {        // alinea a la derecha
    return string(max(0, ancho - largoUtf8(s)), ' ') + s;
}

string num(double x, int dec = 2) {
    ostringstream o;
    o << fixed << setprecision(dec) << x;
    return o.str();
}

double mediana(const vector<double>& v, int ini, int fin) {   // rango [ini, fin)
    int n = fin - ini;
    if (n % 2 == 1) return v[ini + n / 2];
    return (v[ini + n / 2 - 1] + v[ini + n / 2]) / 2.0;
}

void fila(const string& nombre, const string& valor) {
    cout << "│ " << izq(nombre, 24) << " │ " << der(valor, 14) << " │\n";
}

int main() {
#ifdef _WIN32
    SetConsoleOutputCP(65001);                  // arregla tildes y símbolos en Windows
#endif

    int n;
    cout << "¿Cuántos números vas a ingresar? ";
    cin >> n;
    if (n <= 0) {
        cout << "n debe ser mayor que 0." << endl;
        return 1;
    }

    // Arreglo con los n números
    vector<double> datos(n);
    for (int i = 0; i < n; i++) {
        cout << "Número " << i + 1 << ": ";
        cin >> datos[i];
    }

    vector<double> orden = datos;
    sort(orden.begin(), orden.end());

    // ---------- Cálculos ----------
    double minimo = orden.front();
    double maximo = orden.back();
    double q2 = mediana(orden, 0, n);
    double q1 = q2, q3 = q2;
    if (n >= 2) {
        q1 = mediana(orden, 0, n / 2);
        q3 = mediana(orden, (n + 1) / 2, n);
    }

    double suma = 0;
    for (double x : datos) suma += x;
    double media = suma / n;

    double sc = 0;
    for (double x : datos) sc += (x - media) * (x - media);
    double varianza = (n > 1) ? sc / (n - 1) : 0.0;
    double desv = sqrt(varianza);
    double cv = (media != 0) ? (desv / fabs(media)) * 100.0 : 0.0;

    // ---------- Tabla de resultados ----------
    cout << "\n  RESULTADOS EDA\n";
    cout << "┌" << rep("─", 26) << "┬" << rep("─", 16) << "┐\n";
    fila("Estadístico", "Valor");
    cout << "├" << rep("─", 26) << "┼" << rep("─", 16) << "┤\n";
    fila("Cantidad (n)", to_string(n));
    fila("Mínimo", num(minimo));
    fila("Q1", num(q1));
    fila("Q2 (mediana)", num(q2));
    fila("Q3", num(q3));
    fila("Máximo", num(maximo));
    fila("Media", num(media));
    fila("Varianza", num(varianza));
    fila("Desviación estándar", num(desv));
    fila("Coef. de variación", num(cv) + " %");
    cout << "└" << rep("─", 26) << "┴" << rep("─", 16) << "┘\n";

    // ---------- Diagrama de barras ----------
    cout << "\n  DIAGRAMA DE BARRAS\n";
    double maxAbs = max(fabs(minimo), fabs(maximo));
    if (maxAbs == 0) maxAbs = 1;
    for (int i = 0; i < n; i++) {
        int largo = (int)round(fabs(datos[i]) / maxAbs * 40);
        if (largo == 0 && datos[i] != 0) largo = 1;
        cout << setw(3) << i + 1 << " │" << rep("█", largo)
             << " " << num(datos[i]) << "\n";
    }
    cout << "    └" << rep("─", 46) << "\n";

    // ---------- Boxplot con valores atípicos ----------
    double iqr = q3 - q1;
    double limInf = q1 - 1.5 * iqr;
    double limSup = q3 + 1.5 * iqr;

    double bigoteInf = maximo, bigoteSup = minimo;      // bigotes sin atípicos
    vector<double> atipicos;
    for (double x : orden) {
        if (x < limInf || x > limSup) {
            atipicos.push_back(x);
        } else {
            bigoteInf = min(bigoteInf, x);
            bigoteSup = max(bigoteSup, x);
        }
    }

    cout << "\n  BOXPLOT\n";
    const int W = 60;
    double rango = (maximo - minimo == 0) ? 1 : (maximo - minimo);
    auto pos = [&](double x) { return (int)round((x - minimo) / rango * (W - 1)); };

    vector<string> celdas(W, " ");
    for (int i = pos(bigoteInf); i <= pos(bigoteSup); i++) celdas[i] = "─";   // bigotes
    for (int i = pos(q1); i <= pos(q3); i++) celdas[i] = "▒";                 // caja
    celdas[pos(bigoteInf)] = "├";
    celdas[pos(bigoteSup)] = "┤";
    celdas[pos(q2)] = "█";                                                    // mediana
    for (double x : atipicos) celdas[pos(x)] = "●";                           // atípicos

    cout << "  ";
    for (const string& c : celdas) cout << c;
    cout << "\n  " << num(minimo)
         << string(max(1, W - largoUtf8(num(minimo)) - largoUtf8(num(maximo))), ' ')
         << num(maximo) << "\n";
    cout << "\n  Leyenda: ├─┤ bigotes   ▒ caja (Q1 a Q3)   █ mediana   ● atípico\n";

    if (atipicos.empty()) {
        cout << "  No hay valores atípicos.\n";
    } else {
        cout << "  Valores atípicos:";
        for (double x : atipicos) cout << " " << num(x);
        cout << "\n";
    }

    return 0;
}
