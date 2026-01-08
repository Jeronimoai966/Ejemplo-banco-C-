# Ejemplo-banco-C-
visualizado por el profesor juan andres (el andri)
casi no acabo esa mierdaa ajajaj
*#include <iostream>
#include <string>
#include <vector>
#include <fstream>
#include <ctime>
#include <iomanip>
#include <algorithm>

// Simulación simple de hash para PIN (en producción usa bcrypt o similar)
std::string hashPin(const std::string& pin) {
    std::string hashed = "HASH_";
    for (char c : pin) {
        hashed += std::to_string(static_cast<int>(c) * 31 + 17);
        hashed += "_";
    }
    return hashed + std::to_string(std::time(nullptr) % 10000);
}

class CuentaBancaria {
private:
    std::string titular;
    std::string numeroCuenta;
    double saldo;
    std::string pinHashed;

public:
    CuentaBancaria(std::string _titular, std::string _numeroCuenta, double saldoInicial, std::string pin)
        : titular(_titular), numeroCuenta(_numeroCuenta), saldo(saldoInicial), pinHashed(hashPin(pin)) {}

    std::string getNumeroCuenta() const { return numeroCuenta; }
    std::string getTitular() const { return titular; }
    double getSaldo() const { return saldo; }

    bool validarPin(const std::string& pin) const {
        return hashPin(pin) == pinHashed;
    }

    bool depositar(double monto) {
        if (monto > 0) {
            saldo += monto;
            return true;
        }
        return false;
    }

    bool retirar(double monto) {
        if (monto > 0 && saldo >= monto) {
            saldo -= monto;
            return true;
        }
        return false;
    }

    void mostrarInfo() const {
        std::cout << "Titular: " << titular << "\n";
        std::cout << "Cuenta: " << numeroCuenta << "\n";
        std::cout << std::fixed << std::setprecision(2);
        std::cout << "Saldo: $" << saldo << "\n\n";
    }
};

class BancoSeguro {
private:
    std::vector<CuentaBancaria> cuentas;
    std::ofstream logFile;
    const int MAX_INTENTOS = 3;

    void registrarLog(const std::string& mensaje) {
        std::time_t ahora = std::time(nullptr);
        char buffer[100];
        std::strftime(buffer, sizeof(buffer), "%Y-%m-%d %H:%M:%S", std::localtime(&ahora));
        
        std::string entrada = std::string(buffer) + " | " + mensaje;
        std::cout << "[LOG] " << entrada << "\n";
        
        if (logFile.is_open()) {
            logFile << entrada << "\n";
            logFile.flush();
        }
    }

public:
    BancoSeguro() {
        logFile.open("transacciones.log", std::ios::app);
        if (logFile.is_open()) {
            registrarLog("=== Sistema bancario iniciado ===");
        }
    }

    ~BancoSeguro() {
        if (logFile.is_open()) {
            registrarLog("=== Sistema bancario cerrado ===");
            logFile.close();
        }
    }

    // Nueva función: crear cuenta
    bool crearCuenta() {
        std::string titular, numeroCuenta, pin;
        double saldoInicial;

        std::cout << "\n--- CREAR NUEVA CUENTA ---\n";
        std::cout << "Nombre del titular: ";
        std::cin.ignore(); // limpiar buffer
        std::getline(std::cin, titular);

        std::cout << "Número de cuenta (ej. 1004): ";
        std::cin >> numeroCuenta;

        // Verificar si ya existe
        if (buscarCuenta(numeroCuenta) != nullptr) {
            std::cout << "¡Error! Ese número de cuenta ya existe.\n\n";
            registrarLog("Intento fallido de crear cuenta duplicada: " + numeroCuenta);
            return false;
        }

        std::cout << "Saldo inicial ($): ";
        std::cin >> saldoInicial;
        if (saldoInicial < 0) saldoInicial = 0;

        std::cout << "Crear PIN de 4 dígitos: ";
        std::cin >> pin;
        if (pin.length() != 4 || !std::all_of(pin.begin(), pin.end(), ::isdigit)) {
            std::cout << "PIN inválido. Debe ser exactamente 4 números.\n\n";
            return false;
        }

        cuentas.emplace_back(titular, numeroCuenta, saldoInicial, pin);
        registrarLog("Cuenta creada exitosamente: " + numeroCuenta + " - " + titular);
        std::cout << "¡Cuenta creada con éxito!\n";
        std::cout << "Titular: " << titular << " | Cuenta: " << numeroCuenta << " | Saldo inicial: $" << saldoInicial << "\n\n";
        return true;
    }

    void agregarCuenta(const CuentaBancaria& cuenta) {
        cuentas.push_back(cuenta);
        registrarLog("Cuenta precargada: " + cuenta.getNumeroCuenta() + " - " + cuenta.getTitular());
    }

    CuentaBancaria* buscarCuenta(const std::string& numeroCuenta) {
        auto it = std::find_if(cuentas.begin(), cuentas.end(),
            [&numeroCuenta](const CuentaBancaria& c) { return c.getNumeroCuenta() == numeroCuenta; });
        return (it != cuentas.end()) ? &(*it) : nullptr;
    }

    CuentaBancaria* autenticar(const std::string& numeroCuenta) {
        CuentaBancaria* cuenta = buscarCuenta(numeroCuenta);
        if (!cuenta) {
            registrarLog("Intento de acceso a cuenta inexistente: " + numeroCuenta);
            std::cout << "Cuenta no encontrada.\n\n";
            return nullptr;
        }

        int intentos = 0;
        std::string pin;
        while (intentos < MAX_INTENTOS) {
            std::cout << "Ingresa tu PIN (4 dígitos): ";
            std::cin >> pin;

            if (cuenta->validarPin(pin)) {
                registrarLog("Acceso exitoso a cuenta: " + numeroCuenta);
                std::cout << "¡Bienvenido, " << cuenta->getTitular() << "!\n\n";
                return cuenta;
            } else {
                intentos++;
                registrarLog("PIN incorrecto para cuenta: " + numeroCuenta + " (intento " + std::to_string(intentos) + ")");
                std::cout << "PIN incorrecto. Te quedan " << (MAX_INTENTOS - intentos) << " intentos.\n";
            }
        }

        registrarLog("Cuenta bloqueada temporalmente: " + numeroCuenta);
        std::cout << "Demasiados intentos. Cuenta bloqueada temporalmente.\n\n";
        return nullptr;
    }

    bool transferir(CuentaBancaria* origen, CuentaBancaria* destino, double monto) {
        if (origen && destino && origen != destino && origen->retirar(monto)) {
            destino->depositar(monto);
            registrarLog("Transferencia: " + origen->getNumeroCuenta() + " -> " + destino->getNumeroCuenta() + 
                        " | Monto: $" + std::to_string(monto));
            return true;
        }
        return false;
    }
};

int main() {
    BancoSeguro banco;

    // Cuentas de ejemplo
    banco.agregarCuenta(CuentaBancaria("Ana Pérez", "1001", 5000.0, "1234"));
    banco.agregarCuenta(CuentaBancaria("Carlos López", "1002", 2500.0, "5678"));

    int opcion;
    std::string numeroCuenta, cuentaDestinoStr;
    double monto;
    CuentaBancaria* cuentaActual = nullptr;

    do {
        std::cout << "=== BANCO SEGURO ===\n";
        if (!cuentaActual) {
            std::cout << "1. Iniciar sesión\n";
            std::cout << "2. Crear nueva cuenta\n";  // ¡NUEVA OPCIÓN!
            std::cout << "3. Salir\n";
            std::cout << "Elige una opción: ";
            std::cin >> opcion;

            if (opcion == 1) {
                std::cout << "Número de cuenta: ";
                std::cin >> numeroCuenta;
                cuentaActual = banco.autenticar(numeroCuenta);
            } else if (opcion == 2) {
                banco.crearCuenta();
            } else if (opcion == 3) {
                std::cout << "¡Gracias por usar Banco Seguro!\n";
                break;
            }
        } else {
            std::cout << "1. Consultar saldo\n";
            std::cout << "2. Depositar\n";
            std::cout << "3. Retirar\n";
            std::cout << "4. Transferir\n";
            std::cout << "5. Cerrar sesión\n";
            std::cout << "Elige una opción: ";
            std::cin >> opcion;

            switch (opcion) {
                case 1:
                    cuentaActual->mostrarInfo();
                    break;
                case 2:
                    std::cout << "Monto a depositar: $";
                    std::cin >> monto;
                    if (cuentaActual->depositar(monto)) {
                        std::cout << "Depósito exitoso. Saldo: $" << cuentaActual->getSaldo() << "\n\n";
                    } else {
                        std::cout << "Monto inválido.\n\n";
                    }
                    break;
                case 3:
                    std::cout << "Monto a retirar: $";
                    std::cin >> monto;
                    if (cuentaActual->retirar(monto)) {
                        std::cout << "Retiro exitoso. Saldo: $" << cuentaActual->getSaldo() << "\n\n";
                    } else {
                        std::cout << "Saldo insuficiente.\n\n";
                    }
                    break;
                case 4:
                    std::cout << "Cuenta destino: ";
                    std::cin >> cuentaDestinoStr;
                    std::cout << "Monto: $";
                    std::cin >> monto;
                    if (auto* destino = banco.buscarCuenta(cuentaDestinoStr)) {
                        if (banco.transferir(cuentaActual, destino, monto)) {
                            std::cout << "Transferencia exitosa.\n";
                            std::cout << "Saldo actual: $" << cuentaActual->getSaldo() << "\n\n";
                        } else {
                            std::cout << "Falló (saldo insuficiente o misma cuenta).\n\n";
                        }
                    } else {
                        std::cout << "Cuenta destino no existe.\n\n";
                    }
                    break;
                case 5:
                    std::cout << "Sesión cerrada.\n\n";
                    cuentaActual = nullptr;
                    break;
                default:
                    std::cout << "Opción inválida.\n\n";
            }
        }
    } while (true);

    return 0;
}
