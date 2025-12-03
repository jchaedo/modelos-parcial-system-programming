# Checklist para parcial 2

### Archivos

```bash
src/
│   
│   ### FUNDAMENTAL REVISAR
│   
├── sched.c     # --> casi siempre modificamos next_task, init y a veces disable
├── mmu.c       # --> principalmente para el page_fault_handler
├── idt.c       # --> vamos a tener que poner las IDT_ENTRY para toda nueva syscall
├── isr.asm     # --> para las nuevas rutinas que vayamos a implementar
│   
│   ### PEGARLE UN OJO POR LAS DUDAS
│   
├── tss.c       # --> con funciones para pisar la tss
├── gdt.c       # --> por si tenemos que definir una nueva 
├── tasks.c     # --> por si tenemos que cambiar la forma en la que se inicializa una tarea ya que cambió mucho sus comportamientos en una consigna
├── idle.asm    # --> idem tasks, sobre todo si cambian el estado predeterminado de las tareas (actualmente TASK_RUNNABLE)
├── shared.h    # --> principalmente environment
│   
│   ### defines principales
│   
├── task_defines.h
├── tasks.h
├── defines.h
├── types.h
│   
│   ### defines y funciones de tareas
│   
├── tareas  # --> raro que cambie algo pero no imposible
│   ├── syscall.h   # --> casi siempre va a haber que definir las syscalls que nos piden implementar
│   ├── task_prelude.asm    # --> raro pero podríamos tener que especificar como loopea la tarea si cambia la forma de ejecución o se pueden deshabilitar
│   ├── task_lib.h  # --> funciones que tienen disp. todas las tareas pero no son syscalls
│   └── ...
│   
│   ### Otros que casi nunca tocaríamos pero los pongo por las dudas
│   
├── kernel.asm  # --> en el contexto de un parcial sería muy raro que nos pidan que cambiemos algo específico acá y no dar indicaciones de un sistema distinto
├── pic.c
├── screen.c
├── keyboard_input.c    # --> si necesitamos ver el input del usuario podríamos necesitarlo
│   
│   ### Si tengo que cambiar alguno de estos estoy en el muere
│   
├── Makefile
├── a20.asm
├── diskette.img.bz2
├── orga2.py
└── ...
```

### Explicaciones

- No olvidar la interpretación y la idea general
- Recordarle al lector todo detalle que se asume o del cual se depende cuando se hace otra función