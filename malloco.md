### Arquitectura y Organización de Computadoras - DC - UBA 
# Segundo Parcial - Primer Cuatrimestre de 2025


## Notas y asunciones

El kernel implementa un sistema de ``lazy allocation`` en el que las tareas pueden **pedir memoria**, pero el kernel solo la reserva **cuando la tarea intenta acceder** a dicha memoria.

Asumimos que:

- Hay espacio suficiente para todos los posibles elementos reserva_t en los arrays de reservas de c/ tarea ($2^{8}-1$ bytes)

- Según los profes no es necesario contemplar el caso en el que se trata a acceder a datos entre dos páginas. Ej: se accede a 4 bytes entre paginas (dword en rango ``[1022:1026)`` )

- Si la tarea está leyendo una dirección que no le corresponde, el kernel debe:
1. desalojarla inmediatamente
2. asegurarse de que no vuelva a correr
3. marcar la memoria reservada por la misma para que sea liberada
4. saltar a la próxima tarea.

- La liberación de memoria va a estar a cargo de la **tarea de nivel 0**: ``garbage_collector`` --> recorre continuamente las reservas de las tareas en busca de reservas marcadas para liberar.

- Como máximo, una tarea puede tener asignados hasta 4 MB de memoria total. Si intenta reservar más memoria, la syscall deberá devolver NULL. (asumo que cuando el enunciado dice memoria asignada, se refiere solamente a la que se pide a través de malloco, y no cuenta el identity mapping y páginas que tiene la tarea de base)

---

```C
typedef struct {
  uint32_t task_id;
  reserva_t* array_reservas;
  uint32_t reservas_size;
} reservas_por_tarea; 
```

```C
typedef struct {
  uint32_t virt; //direccion virtual donde comienza el bloque reservado
  uint32_t tamanio; //tamaño del bloque en bytes
  uint8_t estado; //0 si la casilla está libre, 1 si la reserva está activa, 2 si la reserva está marcada para liberar, 3 si la reserva ya fue liberada
} reserva_t; 
```

```C
reservas_por_tarea* dameReservas(int task_id);
```

## Ejercicio 1:

Cuando la tarea trate de acceder por primera vez a la memoria en el rango de direcciones virtuales que reservó:
- Va a saltar un page fault, en el cual se va a:
  - verificar si el acceso corresponde a una región de memoria virtual que reservó la tarea (utilizamos esMemoriaReservada, función en la cual recorremos array_reservas buscando si hay un elemento reserva_t que tenga el rango de vaddr que corresponde a la dirección pedida)
    - si la memoria le corresponde a la tarea y no estaba asignada la página física, vamos a pedir una página de usuario, hacerle zero_page y mappearle la vaddr que nos dieron
    - si no: vamos a cambiar el estado de la tarea a TASK_SLOT_FREE para eliminarla permanentemente y marcar toda la memoria que la tarea tenía reservada para que sea liberada por el garbage_collector

  - <div style="color: red">Estaría bien pisar su estado con TASK_SLOT_FREE ? Rta profesor: se puede usar tanto TASK_PAUSED como TASK_SLOT_FREE</div>

---

En resumen:

- Necesitamos guardar las nuevas estructuras reservas_por_tarea y y sus correpondientes reservas en una estructura que pueda ver solamente el garbage_collector. Asumimos que el garbage_collector se inicializa con páginas (de usuario, ya que no tienen por qué verla) donde guarda toda esta información (la mayor cantidad de slots de reserva_t que podría requerir cada tarea). Es decir:
  - Si tenemos n tareas y sabemos que c/u va a guardar como mucho X reservas durante su vida útil.
  - Entonces el garbage_collector tiene ( ``n* ( X* sizeof(reserva_t) + sizeof(reservas_por_tarea) )`` ) bytes mappeados.
- Necesitamos modificar el **page fault handler**. En él vamos a poner el manejo del caso en el que la tarea accede a un area reservada por primera vez o accede a un area inválida (en cuyo caso se la desaloja permanentemente)
- Hay que agregar las entradas de las syscalls ``malloco`` y ``chau``
- Vamos a guardar:
  - El registro de cuánta memoria pidió la tarea (para verificar que no se pase de los 4MB)
  - La dirección que corresponde a la siguiente vaddr que puede reservar (ya que comenzamos desde ``0xA10C0000``)
  
  - Esto lo vamos a hacer agregandole dos atributos más a las ``sched_entry_t`` de ``sched_tasks``


### Algunos archivos principales que modificamos del kernel original (no se cuentan los cambios implicitos del enunciado como, por ejemplo, las tareas que se van a inicializar en el tasks_init)

```bash
src/
│   
├── sched.c     # CAMBIA: sched_tasks, sched_entry_t, sched_next_task
├── mmu.c       # CAMBIA: page_fault_handler
├── idt.c       # CAMBIA: agregamos IDT_ENTRY3(90) e IDT_ENTRY3(91)
├── isr.asm     # CAMBIA: agregamos rutina _isr90 e _isr91
│   
├── tss.c       # CAMBIA: definimos tss_t tss_garbage_collector
├── gdt.c       # no cambia
├── tasks.c     # no cambia
├── idle.asm    # no cambia
├── shared.h    # no cambia
│   
│   ### defines principales
│   
├── task_defines.h  # CAMBIA: agregamos GDT_IDX_TASK_GARBAGE_COLLECTOR
├── tasks.h         # no cambia
├── defines.h       # no cambia
├── types.h         # no cambia
│   
│   ### defines y funciones de tareas
│   
├── tareas 
│   ├── syscall.h               # CAMBIA: agregamos la syscall que llama a la int90 y la que llama a la int91 (malloco y chau)
│   ├── task_prelude.asm        # no cambia
│   ├── task_lib.h              # no cambia
│   ├── task_garbage_collector.c  # NUEVO ARCHIVO
│   └── ...
│   
└── ...
```

### Ahora veamos los cambios puntuales:

#### sched.c     # CAMBIA: sched_tasks, sched_entry_t, sched_next_task

```c
// TODO
// sched_next_task ahora considera el caso en el que hubieron 100 ticks y salta al garbage_collector
```
#### mmu.c       # CAMBIA: page_fault_handler

```c
// TODO
// Se chequea si la dirección pertenece a un rango de memoria reservada. No olvidemos que el enunciado garatiza que la memoria se reserva en múltiplos de 1024 (y sí, esto cuenta el caso en el que se reservan 0 bytes!)
```

#### mmu.c       # CAMBIA: page_fault_handler

```c
// TODO
// hacemos un espacio más para la tarea especial garbage_collector
// #define GDT_IDX_TASK_GARBAGE_COLLECTOR            13
// vamos a tener que desplazar el offset de la gdt que usa el tasks.c en create_task para acceder a los slots de gdt disponibles en los que guarda los task gate
// #define GDT_TSS_START 14
```

## Ejercicio 2:

Para la tarea ``garbage_collector``, vamos a tomarla como una tarea especial. Esta no va a estar en sched_tasks, sino que va a llamarse al selector ``GARBAGE_COLLECTOR_SELECTOR`` en el caso en el que ```environment->ticks_count``` sea múltiplo de 100.

### Creamos el archivo task_garbage_collector.c

garbage_collector es una tarea más, puede ser interrumpida por otra cuando está liberando la memoria. Si una tarea trata de acceder a la memoria cuando el garbage collector todavía no terminó de eliminarla, no es problemático, ya que lo que define si la tarea puede o no acceder a memoria es

#### taskGarbageCollector.c
```c
#include "task_lib.h"

#define RESERVAR_POR_TAREA_STRUCT_START ...

// asumo que tengo reservas_por_tarea_t* info_reservas = RESERVAS_POR_TAREA_ARRAY_START;
// y que a su vez esta tiene inicializados los arrays de reservas de c/ tarea con estado 0 en todas las reservas que no fueron ocupadas

void liberar_reserva(reserva_t *reserva, uint32_t task_cr3) {
      
  // recordamos:
  // uint8_t estado;
  // 0 si la casilla está libre
  // 1 si la reserva está activa
  // 2 si la reserva está marcada para liberar
  // 3 si la reserva ya fue liberada

  if (reserva->estado == 2) {

    uint32_t virt = reserva->virt;
    uint32_t tamanio = reserva->tamanio;
    // vamos a aprovechar que las reservas son múltiplos de 4KB (el tamaño de una página)
    // hacemos una copia de tamanio y vamos a ir liberando todas las páginas que correspondan hasta que el tamanio que nos quede sea 0
    
    while (tamanio > 0) {

      // paddr_t mmu_unmap_page(uint32_t cr3, vaddr_t virt) no hace nada cuando la página ya está desmappeada
      mmu_unmap_page(task_cr3, virt + tamanio);

      tamanio -= PAGE_SIZE;
    }
  }
}

void task(void) {
  size_t i = 0;
	while (true) {

    reserva_t *reservas = info_reservas[i]->array_reservas;
    
    uint32_t cant_reservas = info_reservas[i]->reservas_size; // esto es simplemente para no hacer tantos accesos a memoria. cuando se vuelva a pasar por los arrays de esa tarea, se va a actualizar la cantidad
    // también se podría empezar a iterar a partir de la primera reserva sin liberar, ya que una vez liberada la reserva i, entonces ya no se va a volver a usar ese slot (de acuerdo al enunciado, ya que no nos importa reutilizar la memoria que liberamos)
    // por simplicidad lo escribo de la forma naive

    uint32_t task_id = info_reservas[i]->task_id;
    uint32_t task_cr3 = task_id_to_cr3(task_id);

    for (uint32_t j = 0; j < cant_reservas; j++) {
      
      liberar_reserva(&reservas[j], task_cr3);
    }

    i = (i+1) % MAX_TASKS;
  }
}

// auxiliar:

uint32_t task_id_to_cr3(uint32_t task_id) {
  int16_t task_selector = sched_tasks[task_id];
  gdt_entry_t *tss_descriptor = &gdt[GDT_TSS_START + task_id];
  tss_t *tss = (  (tss_descriptor->base_15_0 << 0)   |
                  (tss_descriptor->base_23_16 << 16) |
                  (tss_descriptor->base_31_24 << 24) );
  return tss->cr3;
}

```

## Ejercicio 3:

a) Indicar dónde podría estár definida la estructura que lleva registro de las reservas (5 puntos)

Por lo que asumí, la estructura está definida en páginas de usuario que tiene el garbage_collector mappeadas. Utilizo páginas de usuario para que las otras tareas no puedan ver la memoria reservada, ya que sólo le corresponde al garbage_collector.

Cuando se hace ``tasks_init``, se puede llamar a una función que mappee las páginas necesarias para poner un array de reservas_por_tarea_t (el cual asumí que existía en el ejercicio anterior, y que estaba en la dirección virtual RESERVAS_POR_TAREA_ARRAY_START). Vamos a llamar al array "info_reservas". Luego, para todo i tq 0 <= i < MAX_TASKS, ``info_reservas[i]->array_reservas`` tiene una dirección mappeada a partir de la cual hay suficiente lugar mappeado para los n * sizeof(reserva_t) bytes, con n la cantidad de reservas que vaya a utilizar cada tarea como máximo.

b) Dar una implementación para `malloco` (10 puntos)

```C
void* malloco(size_t size) {
  

  // TODO
  return NULL;
}
```

```C
void chau(virtaddr_t virt);
```

c) Dar una implementación para `chau` (10 puntos)
