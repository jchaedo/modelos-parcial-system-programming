# Solución

## Parte 1:

1. 

### En defines.h

```C
#define VADDR_COMIENZO_MEM_COMPARTIDA_PAREJA 0xC0C00000
#define VADDR_FIN_MEM_COMPARTIDA_PAREJA 0xC0C00000 + 4 * 1024 * 1024 // + 2^22
```


### Syscalls

Los parámetros de las syscall serán pasados por el registro `ecx`.

#### Cambios en sched.c

```c
typedef enum {
  TASK_SLOT_FREE,
  TASK_RUNNABLE,
  TASK_PAUSED,
  // modificacion: nuevo estado que bloquea a la tarea hasta que encuentra pareja
  TASK_BUSCANDO_PAREJA
} task_state_t;
```

```c
typedef struct {
  int16_t selector;
  task_state_t state;
  // modificacion: agregamos estados para identificar la relación de pareja de cada tarea
  task_estado_pareja_t estado_pareja;
  task_id_t pareja;
} sched_entry_t;
```

```c
// modificacion: definimos un enum para identificar si una tarea está en pareja y qué rol tiene
typedef enum {
    SIN_PAREJA,
    EN_PAREJA_LIDER,
    EN_PAREJA_NO_LIDER
} task_estado_pareja_t;
```

#### Cambios en idt.c

```c
// modificación: Entradas para las nuevas syscalls. todas con permisos de usuario
  IDT_ENTRY3(90);
  IDT_ENTRY3(91);
  IDT_ENTRY3(92);
```

#### Cambios en isr.asm

```asm
; Vamos a utilizar esta interrupción para la syscall
global _isr90
_isr90:
  pushad

  call crear_pareja

  popad
  iret
```
```asm
; Vamos a utilizar esta interrupción para la syscall
global _isr91
_isr91:
  pushad

  push ecx
  call juntarse_con
  add esp, 4

  popad
  iret
```
```asm
; Vamos a utilizar esta interrupción para la syscall
global _isr92
_isr92:
  pushad

  call abandonar_pareja

  popad
  iret
```

### `crear_pareja()`

```C
void crear_pareja() {

  if (!aceptando_pareja(current_task)) return;  // si ya tiene una pareja, ignoramos la solicitud

  // sino:

  sched_entry_t *tarea = &sched_tasks[current_task];
  
  tarea->state = BUSCANDO_PAREJA;     //  cambiamos su estado a uno diferente de TASK_RUNNABLE, por lo que el scheduler no va a saltar a la misma hasta que cambie de estado. queda bloqueada
  tarea->estado_pareja = EN_PAREJA_LIDER;   // definimos su estado. esto luego lo vamos a utilizar para determinar que esta tarea tiene permiso r/w en la memoria compartida por la tarea

  return;
}
```

#### Cambios en sched.c

```c
int8_t sched_add_task(uint16_t selector) {
    ...
        if (sched_tasks[i].state == TASK_SLOT_FREE) {
            sched_tasks[i] = (sched_entry_t) {
            .selector = selector,
            .state = TASK_PAUSED,
            // modificacion: ahora el estado predeterminado va a ser sin pareja
            .estado_pareja = SIN_PAREJA
            .pareja = 0
            };
            return i;
        }
    ...
}
```


### `juntarse_con(id_tarea)`

Para unirse a una pareja una tarea deberá invocar juntarse_con(<líder-de-la-pareja>). Aquí se deben considerar las siguientes situaciones:

```C
int juntarse_con(int id_tarea) {

  if (!aceptando_pareja(current_task)) return 1;  // si la tarea ya tiene una pareja, retornamos 1
  
  if (sched_tasks[id_tarea].state != TASK_BUSCANDO_PAREJA) return 1; // si la tarea id_tarea no estaba creando una pareja entonces retornamos 1

  // sino, se conforma la pareja

  sched_tasks[current_task].state = TASK_RUNNABLE; // ahora la tarea lider puede volver a correr
  sched_tasks[current_task].estado_pareja = EN_PAREJA_NO_LIDER; //  le asignamos la etiqueta corresp. a la tarea actual

  return 0;
}
```

#### Cambios en mmu.c

```c
bool page_fault_handler(vaddr_t virt) {
  print("Atendiendo page fault...", 0, 0, C_FG_WHITE | C_BG_BLACK);
  // Chequeemos si el acceso fue dentro del area on-demand
  if (virt >= ON_DEMAND_MEM_START_VIRTUAL && virt < ON_DEMAND_MEM_END_VIRTUAL) {
    // En caso de que si, mapear la pagina
    mmu_map_page(rcr3(), virt, ON_DEMAND_MEM_START_PHYSICAL, MMU_P | MMU_U | MMU_W);
    return true;
    
    // modificación: caso en el que una tarea trata de acceder a direcciones virtuales corresp a la mem de pareja
  } else if ( VADDR_COMIENZO_MEM_COMPARTIDA_PAREJA <= virt &&
              virt < VADDR_FIN_MEM_COMPARTIDA_PAREJA &&
              !aceptando_pareja(current_task)) {  // si la tarea accedió a memoria compartida de pareja y está emparejada

    // como es la primera vez que se accede, se tiene que mapear a ambar tareas
    paddr_t pagina_paddr = mmu_next_free_user_page();

    int8_t task_id_lider      =  es_lider(current_task) ? current_task : pareja_de_actual();
    int8_t task_id_no_lider   = !es_lider(current_task) ? current_task : pareja_de_actual();

    // con mmu_map_page la pagina queda con ceros porque internamente usa zero_page(page)
    mmu_map_page(task_id_to_cr3(task_id_lider)    , virt, pagina_paddr, MMU_P | MMU_U | MMU_W );
    mmu_map_page(task_id_to_cr3(task_id_no_lider) , virt, pagina_paddr, MMU_P | MMU_U         );
    return true;
  }
  return false;  
}
```
```c
uint32_t task_id_to_cr3(task_id id) {

  uint16_t tss_desc_idx = sched_tasks[id].selector >> 3;
  gdt_entry_t tss_desc = gdt[tss_desc_idx];
  
  tss_t* tss = (tss_t*)( (tss_desc.base_15_0  << 0)  |
                        (tss_desc.base_23_16 << 16) |
                        (tss_desc.base_31_24 << 24) );

  return tss->cr3;
}

```

### `abandonar_pareja()`


- Si la tarea pertenece a una pareja y **no es su líder**: La tarea abandona la pareja, por lo que pierde acceso al área de memoria especificada.
- Si la tarea pertenece a una pareja y **es su líder**: La tarea queda bloqueada hasta que la otra parte de la pareja abandone. Luego pierde acceso a al área de memoria especificada.

```C
void abandonar_pareja() {

  if (aceptando_pareja(current_task)) return;

  if (es_lider(current_task)) {

    
      mmu_unmap_range(task_id_to_cr3(task_id id), area_pareja_compartida)
      shed_tasks[current_task]->state = TASK_LIDER_SOLO
      shed_tasks[current_task]->state = TASK_LIDER_SOLO

      // pendiente de terminar...
  } else {

  }

}
```


### Funciones aux

- `task_id pareja_de_actual()`: si la tarea actual está en pareja devuelve el `task_id` de su pareja, o devuelve 0 si la tarea actual no está en pareja.

```c
task_id pareja_de_actual() {

    if( aceptando_pareja(current_task) ) return 0;

    return id_de_pareja(current_task);  // a implementar acá
}

```

- `bool es_lider(task_id tarea)`: indica si la tarea pasada por parámetro es lider o no.

```c
bool es_lider(task_id tarea) {

    return (sched_tasks[tarea].state == EN_PAREJA_LIDER);
}

```

- `bool aceptando_pareja(task_id tarea)`: si la tarea pasada por parámetro está en un estado que le permita formar pareja devuelve 1, si no devuelve 0.

```c
bool aceptando_pareja(task_id tarea) {

    return (sched_tasks[tarea].estado_pareja == SIN_PAREJA);
}

```

- `void conformar_pareja(task_id tarea)`: informa al sistema que la tarea actual y la pasada por parámetro deben ser emparejadas. Al pasar `0` por parámetro, se indica al sistema que la tarea actual está disponible para ser emparejada.

```c
void conformar_pareja(task_id tarea) {

}
```

- `void romper_pareja()`: indica al sistema que la tarea actual ya no pertenece a su pareja actual. Si la tarea actual no estaba en pareja, no tiene efecto.

```c
void romper_pareja() {

}

```