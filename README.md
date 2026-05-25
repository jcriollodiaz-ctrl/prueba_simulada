from abc import ABC, abstractmethod
from datetime import datetime


# ==============================
# MANEJO DE LOGS
# ==============================

def registrar_log(mensaje):
    with open("sistema_logs.txt", "a", encoding="utf-8") as archivo:
        archivo.write(f"{datetime.now()} - {mensaje}\n")


# ==============================
# EXCEPCIONES PERSONALIZADAS
# ==============================

class SistemaError(Exception):
    pass


class DatoInvalidoError(SistemaError):
    pass


class ServicioNoDisponibleError(SistemaError):
    pass


class ReservaError(SistemaError):
    pass


# ==============================
# CLASE ABSTRACTA GENERAL
# ==============================

class EntidadSistema(ABC):
    def __init__(self, codigo):
        if not codigo:
            raise DatoInvalidoError("El código no puede estar vacío.")
        self._codigo = codigo

    @abstractmethod
    def mostrar_info(self):
        pass


# ==============================
# CLASE CLIENTE
# ==============================

class Cliente(EntidadSistema):
    def __init__(self, codigo, nombre, documento, correo):
        super().__init__(codigo)

        if len(nombre.strip()) < 3:
            raise DatoInvalidoError("El nombre del cliente debe tener mínimo 3 caracteres.")

        if not documento.isdigit():
            raise DatoInvalidoError("El documento solo debe contener números.")

        if "@" not in correo or "." not in correo:
            raise DatoInvalidoError("El correo electrónico no es válido.")

        self.__nombre = nombre
        self.__documento = documento
        self.__correo = correo

    def mostrar_info(self):
        return f"Cliente: {self.__nombre} | Documento: {self.__documento} | Correo: {self.__correo}"

    def get_nombre(self):
        return self.__nombre


# ==============================
# CLASE ABSTRACTA SERVICIO
# ==============================

class Servicio(EntidadSistema):
    def __init__(self, codigo, nombre, precio_base, disponible=True):
        super().__init__(codigo)

        if precio_base <= 0:
            raise DatoInvalidoError("El precio base debe ser mayor que cero.")

        self.nombre = nombre
        self.precio_base = precio_base
        self.disponible = disponible

    @abstractmethod
    def calcular_costo(self, duracion, impuesto=0, descuento=0):
        pass

    @abstractmethod
    def describir_servicio(self):
        pass

    def validar_disponibilidad(self):
        if not self.disponible:
            raise ServicioNoDisponibleError(f"El servicio {self.nombre} no está disponible.")


# ==============================
# SERVICIOS ESPECIALIZADOS
# ==============================

class ReservaSala(Servicio):
    def calcular_costo(self, duracion, impuesto=0, descuento=0):
        if duracion <= 0:
            raise DatoInvalidoError("La duración debe ser mayor que cero.")

        subtotal = self.precio_base * duracion
        total = subtotal + (subtotal * impuesto) - descuento

        if total <= 0:
            raise DatoInvalidoError("El costo total no puede ser menor o igual a cero.")

        return total

    def describir_servicio(self):
        return "Reserva de sala para reuniones, clases o capacitaciones."

    def mostrar_info(self):
        return f"Servicio: {self.nombre} | Tipo: Reserva de sala | Precio hora: {self.precio_base}"


class AlquilerEquipo(Servicio):
    def calcular_costo(self, duracion, impuesto=0, descuento=0):
        if duracion <= 0:
            raise DatoInvalidoError("La duración del alquiler debe ser mayor que cero.")

        subtotal = self.precio_base * duracion
        seguro = 20000
        total = subtotal + seguro + (subtotal * impuesto) - descuento
        return total

    def describir_servicio(self):
        return "Alquiler de equipos tecnológicos como computadores o proyectores."

    def mostrar_info(self):
        return f"Servicio: {self.nombre} | Tipo: Alquiler de equipo | Precio hora: {self.precio_base}"


class AsesoriaEspecializada(Servicio):
    def calcular_costo(self, duracion, impuesto=0, descuento=0):
        if duracion <= 0:
            raise DatoInvalidoError("La duración de la asesoría debe ser mayor que cero.")

        subtotal = self.precio_base * duracion
        valor_profesional = 50000
        total = subtotal + valor_profesional + (subtotal * impuesto) - descuento
        return total

    def describir_servicio(self):
        return "Asesoría especializada en tecnología, software o gestión empresarial."

    def mostrar_info(self):
        return f"Servicio: {self.nombre} | Tipo: Asesoría especializada | Precio hora: {self.precio_base}"


# ==============================
# CLASE RESERVA
# ==============================

class Reserva:
    def __init__(self, cliente, servicio, duracion):
        if not isinstance(cliente, Cliente):
            raise ReservaError("El cliente no es válido.")

        if not isinstance(servicio, Servicio):
            raise ReservaError("El servicio no es válido.")

        if duracion <= 0:
            raise ReservaError("La duración de la reserva debe ser mayor que cero.")

        self.cliente = cliente
        self.servicio = servicio
        self.duracion = duracion
        self.estado = "Pendiente"

    def confirmar(self):
        try:
            self.servicio.validar_disponibilidad()
            costo = self.servicio.calcular_costo(self.duracion, impuesto=0.19, descuento=10000)
        except ServicioNoDisponibleError as error:
            registrar_log(f"ERROR DE DISPONIBILIDAD: {error}")
            raise ReservaError("No se pudo confirmar la reserva.") from error
        except DatoInvalidoError as error:
            registrar_log(f"ERROR DE DATOS EN RESERVA: {error}")
            raise
        else:
            self.estado = "Confirmada"
            registrar_log(f"Reserva confirmada para {self.cliente.get_nombre()} por valor de ${costo}")
            return costo
        finally:
            registrar_log("Proceso de confirmación finalizado.")

    def cancelar(self):
        if self.estado == "Cancelada":
            raise ReservaError("La reserva ya se encuentra cancelada.")

        self.estado = "Cancelada"
        registrar_log(f"Reserva cancelada para {self.cliente.get_nombre()}")

    def procesar(self):
        try:
            costo = self.confirmar()
            print(f"Reserva procesada correctamente. Costo total: ${costo:,.0f}")
        except SistemaError as error:
            print(f"No se pudo procesar la reserva: {error}")
            registrar_log(f"ERROR AL PROCESAR RESERVA: {error}")

    def mostrar_reserva(self):
        return f"Cliente: {self.cliente.get_nombre()} | Servicio: {self.servicio.nombre} | Estado: {self.estado}"


# ==============================
# SIMULACIÓN DE OPERACIONES
# ==============================

def ejecutar_operacion(numero, descripcion, funcion):
    print(f"\nOperación {numero}: {descripcion}")
    try:
        resultado = funcion()
    except SistemaError as error:
        print(f"Error controlado: {error}")
        registrar_log(f"Operación {numero} fallida: {error}")
    except Exception as error:
        print(f"Error inesperado: {error}")
        registrar_log(f"Error inesperado en operación {numero}: {error}")
    else:
        if resultado:
            print(resultado)
        registrar_log(f"Operación {numero} ejecutada correctamente.")
    finally:
        print("Operación finalizada.")


def main():
    print("SISTEMA INTEGRAL DE GESTIÓN - SOFTWARE FJ")

    clientes = []
    servicios = []

    ejecutar_operacion(1, "Registrar cliente válido", lambda: clientes.append(
        Cliente("C001", "Jhon Edwin Criollo Diaz", "120367387", "jhon@gmail.com")
    ) or "Cliente registrado correctamente.")

    ejecutar_operacion(2, "Registrar cliente con documento inválido", lambda:
        Cliente("C002", "Ana López", "ABC123", "ana@gmail.com")
    )

    ejecutar_operacion(3, "Registrar cliente con correo inválido", lambda:
        Cliente("C003", "Carlos Díaz", "987654", "correo_invalido")
    )

    ejecutar_operacion(4, "Crear servicio reserva de sala", lambda: servicios.append(
        ReservaSala("S001", "Sala de reuniones", 50000)
    ) or "Servicio creado correctamente.")

    ejecutar_operacion(5, "Crear servicio alquiler de equipo", lambda: servicios.append(
        AlquilerEquipo("S002", "Alquiler de computador", 40000)
    ) or "Servicio creado correctamente.")

    ejecutar_operacion(6, "Crear servicio asesoría especializada", lambda: servicios.append(
        AsesoriaEspecializada("S003", "Asesoría en software", 80000)
    ) or "Servicio creado correctamente.")

    ejecutar_operacion(7, "Crear servicio con precio inválido", lambda:
        ReservaSala("S004", "Sala económica", -20000)
    )

    ejecutar_operacion(8, "Reserva exitosa de sala", lambda:
        Reserva(clientes[0], servicios[0], 3).procesar()
    )

    ejecutar_operacion(9, "Reserva fallida por duración inválida", lambda:
        Reserva(clientes[0], servicios[1], 0).procesar()
    )

    def reserva_servicio_no_disponible():
        servicio_no_disponible = AsesoriaEspecializada("S005", "Asesoría no disponible", 90000, False)
        reserva = Reserva(clientes[0], servicio_no_disponible, 2)
        reserva.procesar()

    ejecutar_operacion(10, "Reserva fallida por servicio no disponible", reserva_servicio_no_disponible)

    ejecutar_operacion(11, "Cancelar una reserva correctamente", lambda: cancelar_reserva(clientes[0], servicios[2]))


def cancelar_reserva(cliente, servicio):
    reserva = Reserva(cliente, servicio, 1)
    reserva.confirmar()
    reserva.cancelar()
    return reserva.mostrar_reserva()


if __name__ == "__main__":
    main()
