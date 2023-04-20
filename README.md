# Puente-de-Ambite





#CÓDIGO CON EXPLICACIONES DE CADA PASO QUE SE REALIZA








import time
import random
from multiprocessing import Lock, Condition, Process
from multiprocessing import Value

#Definimos las variables norte y sur
SOUTH = 1
NORTH = 0

#Definimos el número de vehiculos y de peatones
NCARS = 10
NPED = 10
TIME_CARS = 0.5  # a new car enters each 0.5s
TIME_PED = 1 # a new pedestrian enters each 5s
TIME_IN_BRIDGE_CARS = (1, 0.5) # normal 1s, 0.5s
TIME_IN_BRIDGE_PEDESTRGIAN = (3, 1) # normal 1s, 0.5s



#Creamos la clase del monitor
class Monitor():
    #Inicializamos la clase
    def __init__(self):
        
        #Definimos un Lock (semáforo binario)
        self.mutex = Lock()
        
        #Definimos una variable Value
        self.patata = Value('i', 0)
        
        #Definimos 2 variables para controlar cuantos coches y cuantos peatones hay dentro del puente
        self.personas_dentro = Value('i', 0)
        self.coches_dentro = Value('i', 0)
        
        #Definimos variables de condicion para controlar si hay coches/puentes en el puente
        self.hay_coches = Condition(self.mutex)
        self.hay_personas = Condition(self.mutex)
        
        #Se elige una preferencia inicial (para controlar los coches que vienen del norte y sur. Mas explicado en el pdf adjunto)
        self.preferencia = 0
        
        #Definimos una variable de condición para ver si hay coches esperando 
        self.hay_coches_esperando = Condition(self.mutex)

        #Definimos 2 contadores, que se van a corresponder a la cola de la fila norte y a la cola de la fila sur 
        self.hay_coches_esperando_norte = 0
        self.hay_coches_esperando_sur = 1

    #Definimos una función para los coches que quieren entrar al puente
    def wants_enter_car(self, direction: int) -> None:
        
        #Primero se bloquea el Lock 
        self.mutex.acquire()
        self.patata.value += 1

        #Si el coche viene del norte
        if direction == 0:
            #Se añade 1 a la cola de coches del norte
            self.hay_coches_esperando_norte += 1
            #Se espera a que no haya personas ni coches dentro
            self.hay_personas.wait_for(self.no_hay_personas_dentro)
            self.hay_coches.wait_for(self.no_hay_coches_dentro)
            #Se espera a que tengan preferencia los coches del Norte
            self.hay_coches_esperando.wait(self.preferencia == 0)
                        
            #Si se puede pasar, se elimina 1 a la cola de coches del norte y se añade 1 a los coches dentro del puente
            self.hay_coches_esperando_norte -= 1
            self.coches_dentro.value += 1


        else: #anñalogo si viene del sur
            self.hay_coches_esperando_sur += 1
            self.hay_personas.wait_for(self.no_hay_personas_dentro)
            self.hay_coches.wait_for(self.no_hay_coches_dentro)
            self.hay_coches_esperando.wait(self.preferencia == 1)
            self.hay_coches_esperando_sur -= 1
            self.coches_dentro.value += 1

            
        #Se libera el semáforo
        self.mutex.release()

    #Definimos una función para cuando un coche quiere salir del puente
    def leaves_car(self, direction: int) -> None:
        
        #Se bloquea el Lock
        self.mutex.acquire() 
        self.patata.value += 1
        
        #Se elimina 1 a los coches dentro del puente
        self.coches_dentro.value -= 1
        #Se libera el semáforo de los coches dentro del puente
        self.hay_coches.notify()
        
        #Se cambia la dirección de preferencia en el caso de que haya coches esperando por la otra dirección
        if direction == 0:
            if self.hay_coches_esperando_sur > 0:
                self.preferencia = 1
        else:
            if self.hay_coches_esperando_norte > 0:
                self.preferencia = 0
                
        #Se libera el semáforo
        self.mutex.release()

    #Definimos una función para cuando un peaton quiere entrar al puente
    def wants_enter_pedestrian(self) -> None:
        #Se bloquea el Lock
        self.mutex.acquire()
        self.patata.value += 1

        #Primero ve si no hay coches dentro del puente
        self.hay_coches.wait_for(self.no_hay_coches_dentro)
        
        #Añade 1 al número de personas dentro del puente
        self.personas_dentro.value += 1
        
        #Se libera el semáforo
        self.mutex.release()

    #Definimos una función para cuando un peatón quiere salir del puente
    def leaves_pedestrian(self) -> None:
        #Se bloquea el Lock
        self.mutex.acquire()
        self.patata.value += 1

        #Reduce en 1 al número de personas dentro del puente
        self.personas_dentro.value -= 1
        #Si no hay personas dentro del puente, libera el semáforo
        if self.personas_dentro.value == 0:
            self.hay_personas.notify()
    
        #Se libera el semáforo
        self.mutex.release()

    #Funcion booleana para ver si no hay personas dentro del puente
    def no_hay_personas_dentro(self):
        return self.personas_dentro.value == 0
    
    #Funcion booleana para ver si no hay coches dentro del puente
    def no_hay_coches_dentro(self):
        return self.coches_dentro.value == 0


#HASTA AQUÍ ES LO QUE HE PROGRAMADO YO, LO QUE VENIA DESPUÉS YA ESTABA AÑADIDO EN LA PRÁCTICA
