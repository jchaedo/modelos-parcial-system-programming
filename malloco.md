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
├── isr.asm     # CAMBIA: agregamos rutina _isr90 e _isr91, cambiamos _isr14 para que desaloje la tarea que accedió a memoria no reservada
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
bool page_fault_handler(vaddr_t virt) {
  
  ...

  if (ON_DEMAND_MEM_START_VIRTUAL <= virt && virt < ON_DEMAND_MEM_END_VIRTUAL) {
    
    ...

  } else if (esMemoriaReservada(virt)) {  // CAMBIO MALLOCO: accede a una dirección reservada por primera vez

    // vamos a mappear la página de usuario correspondiente a la dirección virtual

    mmu_map_page(rcr3(), virt, mmu_next_free_user_page(), MMU_P | MMU_W | MMU_U);
  } 
  
  // si llegamos acá, significa que la tarea trató de acceder a memoria que no le corresponde

  // así que devolvemos false y vamos a utilizar eso en _isr14 para desalojar la tarea y evitar que vuelva a correr
  return false;
}
```

#### isr.asm       # CAMBIA: _isr14

```
extern current_task_kill
...

global _isr14
_isr14:
  pushad
  mov eax, cr2
  push eax
  call page_fault_handler
  pop ecx
  cmp al, 0

  ; acá cambiamos el comportamiento
  ; cuando cuando el page_fault_handler devuelva false, vamos a saltar a acceso_a_mem_invalido

  je .acceso_a_mem_invalido

  popad
	add esp, 4 ; error code
	iret
.acceso_a_mem_invalido:
  
  ; vamos a desalojar la tarea y liberar toda la memoria que tiene mappeada, luego devolvemos la siguiente tarea a ejecutar y saltamos a ella
  call current_task_kill
  
  mov word [sched_task_selector], ax
  jmp far [sched_task_offset]

  jmp $
```

```c
uint16_t current_task_kill(void) {

  reservas_por_tarea* reservas = dameReservas(current_task);
  reserva_t *arrayReservas = reservas->array_reservas;

  for (size_t i = 0; i < reservas->reservas_size; i++) {

    liberar_reserva(&arrayReservas[i], rcr3());
  }
  // ya que estamos, actualizamos la cant de memoria que reservó la tarea.
  sched_tasks[current_task].cuantoReservo = 0;

  sched_task_disable(current_task);
  
  return sched_next_task();
}
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

    // nos guardamos el array de reservas de la tarea i
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

Por lo que asumí, la estructura está definida en páginas de kernel. Esta estructura está en la dirección virtual RESERVAS_POR_TAREA_ARRAY_START (que también es la física porque tenemos identity mapping).

Cuando se hace ``tasks_init``, se puede llamar a una función que inicialice las estructuras necesarias para poner gestionar la memoria que se reserva. Esto sería:
  - Un array de reservas_por_tarea_t (el cual asumí que existía en el ejercicio anterior, y que estaba en la dirección virtual RESERVAS_POR_TAREA_ARRAY_START). Lo llamamos ``info_reservas``.
  - Luego, para todo i t.q. 0 <= i < MAX_TASKS,
    - ``info_reservas[i]->array_reservas`` tiene una dirección mappeada a partir de la cual hay suficiente lugar para los ``n * sizeof(reserva_t)`` bytes, con n la cantidad de reservas que vaya a utilizar cada tarea como máximo.

b) Dar una implementación para `malloco` (10 puntos)

```C
#define TASK_MAX_BYTES_RESERVABLES (4 * 1024 * 1024) // 2^2 * 2^20 = 4MB

void* malloco(size_t size) {

  // primero vamos a verificar si la tarea puede reservar más memoria
  uint32_t cuantoReservo = sched_tasks[current_task].cuantoReservo;

  if (cuantoReservo + size < TASK_MAX_BYTES_RESERVABLES) {
    
    uint32_t vaddr_a_devolver = sched_tasks[current_task].sig_vaddr;

    // vamos a acceder a la estructura que tiene los datos de las reservas de la tarea actual

    reservas_por_tarea_t *reservas = dameReservas(current_task);  // aprovechamos la función provista por la cátedra

      // como dijimos antes, asumimos que tenemos suficiente espacio en el array y que ya está reservada toda la memoria
      uint32_t tamReservas = reservas->reservas_size;

      reservas->array_reservas[tamReservas] = (reserva_t) {.virt = vaddr_a_devolver,
                                              .tamanio = size,
                                              .estado = 1};
      // actualizamos la cantidad de elem en el array
      reservas->reservas_size++;

      // actualizamos la proxima vaddr de la tarea (recordar que asumimos que los tamaños de reservas son múltiplos de PAGE_SIZE)
      sched_tasks[current_task]->sig_vaddr += size;

      return (void *) vaddr_a_devolver;
  }
  
  // si llegamos acá es que no podía reservar más
  return NULL;
}
```


c) Dar una implementación para `chau` (10 puntos)

```C
void chau(virtaddr_t virt) {

  reservas_por_tarea_t *reservas = dameReservas(current_task);
  reserva_t *arrayReservas = reservas->array_reservas;
  
  // Vamos a buscar si hay una reservas en el array de la tarea que tenga esa dirección virtual
  // y que esté marcada como "activa", ya que si ya se liberó o la casilla está libre, no debemos modificarla

  for (size_t i = 0; i < reservas->reservas_size; i++) {
    if (arrayReservas[i]->virt == virt && arrayReservas[i]->estado == 1) {

      arrayReservas[i]->estado = 3; // lo cambiamos para que el garbage_collector lo elimine
      break;  // cortamos porque, por la forma en gestionamos las reservas sin reciclar, solo va a haber una reserva con esa vaddr y ya no tiene sentido iterar
    }
  }
}
```