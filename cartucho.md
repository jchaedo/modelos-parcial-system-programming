
#### defines.h

```c

#define BUFFER_CARTUCHO_PADDR 0xF151C000
#define DMA_VADDR 0xBABAB000 // se mappean las tareas que tienen dma desde esta dir a buffer
#define TIPO_ACCESO_VADDR 0xACCE5000  // aca guadamos el modo de acceso de c/tarea

typedef enum {
    NO_ACCESS,
    DMA_ACCESS,
    COPY_ACCESS
} acceso_t;

```

---

#### sched.c

```c

typedef enum {
  TASK_SLOT_FREE,
  TASK_RUNNABLE,
  TASK_PAUSED,
  // nuevo estado usado por cartucho: tarea copiando buffer
  TASK_BLOCKED
} task_state_t;
```

```c
typedef struct {
  int16_t selector;
  task_state_t state;
  // nueva info de tarea para definir cómo copia y adónde
  vaddr_t copy_vaddr;
  uint8_t mode;
} sched_entry_t;

```

```c
int8_t sched_add_task(uint16_t selector) {
  kassert(selector != 0, "No se puede agregar el selector nulo");

  // Se busca el primer slot libre para agregar la tarea
  for (int8_t i = 0; i < MAX_TASKS; i++) {
    if (sched_tasks[i].state == TASK_SLOT_FREE) {
      sched_tasks[i] = (sched_entry_t) {
        .selector = selector,
        .state = TASK_PAUSED,
        // atrib nuevos
        .copy_vaddr = 0x0,     // no tiene sentido este valor si no hay acceso
        .mode = NO_ACCESS,
      };
      return i;
    }
  }
  kassert(false, "No task slots available");
}
```

---

#### idt.c

```c

void idt_init() {
  // Excepciones
  ...

  // Interrupciones de reloj y teclado
  ...

  //
  IDT_ENTRY0(40);   // generamos la entrada de la idt para la int 40 del cartucho

  // Syscalls
  IDT_ENTRY3(90);
  IDT_ENTRY3(91);
  ...
}
```

---

#### isr.asm

```asm

global _isr40

_isr40:
  pushad

  call pic_finish
  call deviceready
  
  popad
  iret
```

```asm
extern opendevice

global _isr90

_isr90:
  pushad
  
  push ecx
  call opendevice

  call sched_next_task

  str cx
  cmp ax, cx
  je .fin
  
  mov word [sched_task_selector], ax
  jmp far [sched_task_offset]
  
  .fin:

  popad
  iret
```

```asm
extern closedevice

global _isr91

_isr91:
  pushad
  
  call closedevice
  
  popad
  iret
```


#### Sin def. seccion


// Funcionamiento: Cuando el buffer esté lleno y listo para proc., el lector informa al kernel con la int IRQ 40 del kernel.

// Copia:

// Copia del buffer en una pdir específica y se mapea en vdir provista por la tarea. 
// Cada tarea debe tener una copia única.

// la dir está dada por ecx al llamar opendevice, y sus permisos van a ser r/w

```c
void opendevice(vaddr_t vaddr) {
    
    sched_tasks[current_task].state = BLOCKED;
    sched_tasks[current_task].mode = (acceso_t)(*TIPO_ACCESO_VADDR);
    sched_tasks[current_task].copy_vaddr = vaddr;
}
```

```c
void closedevice() {
    
    if (sched_tasks[current_task].mode = DMA_ACCESS) {
      
      mmu_unmap_page(rcr3(), (paddr_t)DMA_VADDR);
    }
    if (sched_tasks[current_task].mode = COPY_ACCESS) {

      mmu_unmap_page(rcr3(), (vaddr_t)sched_tasks[current_task].copy_addr);
    }

    sched_tasks[current_task].mode = NO_ACCESS;
}
```

```c
void deviceready() {
    
    for (i = 0; i < MAX_TASKS; i++) {

      sched_entry_t *task = &sched_tasks[i];

      if (task->mode != NO_ACCESS) {

        if (task->state == TASK_BLOCKED) {

          if (task->mode == DMA_ACCESS) {
            
            buffer_dma(CR3_TO_PAGE_DIR(get_task_cr3(task->selector)));

          }
          if (task->mode == COPY_ACCESS) {

            // necesitamos crear una pagina que copie el contenido del buffer

            buffer_copy(CR3_TO_PAGE_DIR(task_selector_to_cr3(task->selector))
                        mmu_next_user_page(),
                        task->copy_addr);
          }
          
          task->state = TASK_RUNNABLE;
        } else {  // si la tarea no está bloqueada, entonces significa que no está pidiendo acceso por primera vez con DMA_ACCESS o COPY_ACCESS. Y como DMA_ACCESS solo tiene que mappear la pagina la primera vez, entonces estamos en COPY_ACCESS y hay que actualizar 
          paddr_t dst_paddr = vaddr_to_paddr(task_selector_to_cr3(task->selector), task->copy_addr);

          copy_page(dst_paddr, BUFFER_CARTUCHO_PADDR);
        }
      }
    }
}
```

```c
// dado el page directory de una tarea realice el mapeo del buffer en modo DMA.
void buffer_dma(pd_entry_t* pd) {
    
    mmu_map_page((uint32_t)pd, (vaddr_t)DMA_VADDR, (paddr_t)BUFFER_CARTUCHO_PADDR, MMU_P | MMU_U);
}
```

```c
// dado el page directory de una tarea realice la copia del buffer a la dirección física pasada por parámetro y realice el mapeo a la dirección virtual pasada por parámetro.
void buffer_copy(pd_entry_t* pd, paddr_t phys, vaddr_t virt) {

    mmu_map_page((uint32_t)pd, virt, phys, MMU_P | MMU_U | MMU_W);
    copy_page(paddr_t, (paddr_t)BUFFER_CARTUCHO_PADDR);
}
```

#### Funciones auxiliares

```c
pd_entry_t* task_selector_to_cr3(uint16_t selector) {
  uint16_t gdt_index = selector >> 3;
  
  gdt_entry_t *task_descriptor = &gdt[gdt_index];

  tss_t *tss = (tss_t *)( (task_descriptor->base_15_0  << 0)  |
                          (task_descriptor->base_23_16 << 16) |
                          (task_descriptor->base_31_24 << 24));
  return tss->cr3;
}
```

```c
void vaddr_to_paddr(uint32_t cr3, vaddr_t virt) {

  uint32_t pd_index = VIRT_PAGE_DIR(virt);
  uint32_t pt_index = VIRT_PAGE_TABLE(virt);
  
  pd_entry_t* pd = CR3_TO_PAGE_DIR(cr3);

  pt_entry_t* pt = pd[pd_index].pt << 12;
  
  paddr_t paddr = pt[pt_index].page << 12;
  
  return paddr;
}
```

#### Consultas:

- Por qué la sol propuesta no cambia sched_add_task ?:
  - En realidad se podría poner un default, es un detalle menor

- En void opendevice(vaddr_t vaddr): `sched_tasks[current_task].mode = (acceso_t)(*TIPO_ACCESO_VADDR);` está bien? en el resuelto usan `sched_tasks[current_task].mode = *(uint8_t*)0xACCE5000;`

- Pedir revisión open/close device