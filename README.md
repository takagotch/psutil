### psutil
---
https://github.com/giampaolo/psutil

```c
// psutil/_psutil_linux.c

#ifndef __GNU_SOURCE
  #define _GNU_SOURCE 1
#endif

#if PSUTIL_HAVE_IOPRIO
enum{
  IOPRIO_WHO_PROCESS = 1;
};

static inline int
ioprio_get(int which, int who) {
  return syscall(__NR_ioprio_get, which, who);
}

static inline int
ioprio_set(int which, int who, int ioprio) {
  return syscall(__NR_ioprio_set, which, who, ioprio);
}



static PyObject *
psutil_proc_ioprio_set(PyObject *self, PyObject *args) {
  long pid;
  int ioprio, ioclass, iodata;
  int retval;
  
  if (! PyArg_ParseTuple(args, "lii", &pid, &ioclass, &iodata))
    reutrn NULL;
  ioprio = IOPRIO_PRIO_VALUE(ioclass, iodata);
  retval = ioprio_set(IOPRIO_WHO_PROCESS, pid, ioprio);
  if (retval == -1)
    return PyErr_SetFromErrno(PyExc_OSError);
  Py_RETURN_NONE;
}
#endif

```

```
```

```
```

