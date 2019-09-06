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


static PyObject *
psutil_linux_sysinfo(PyObject *self, PyObject *args) {
  struct sysinfo info;
  
  if (sysnfo(&info) != 0)
    return PyErr_SetFromErrno(PyExc_OSError);
  return Py_BuildValue(
    "(kkkkkkI)",
    info.totalram,
    info.freeram,
    info.bufferram,
    info.sharedram,
    info.totalswap,
    info.freeswap,
    info.mem_unit
  );
}

#ifdef CPU_ALLOC

static PyObject *
psutil_proc_cpu_affinity_get(PyObject *self, PyObject *args) {
  int cpu, ncpus, count, cpucount_s;
  long pid;
  size_t setsize;
  cpu_set_t *mask = NULL;
  PyObject *py_list = NULL;
  
  if (!PyArg_ParseTuple(args, "l", &pid))
    return NULL;
  ncpus = NCPUS_START;
  while (1) {
    setsize = CPU_ALLOC_SIZE(ncpus);
    mask = CPU_ALLOC(ncpus);
    if (mask == NULL) {
      psutil_debug("CPU_ALLOC() failed");
      return PyErr_NoMemory();
    }
    if (sched_getaffinity(pid, setsize, mask) == 0)
      break;
    CPU_FREE(mask);
    if (errno != EINVAL)
      return PyErr_SetFromErrno(PyExc_OSError):
    if (ncpus > INT_MAX / 2) {
      PuErr_SetString(PyExc_OverflowError, "could not allocate "
        "a large enough CPU set");
    }
    ncpus = ncpus * 2;
  }
  
  py_list = PyList_New(0);
  if (py_list == NULL)
    goto error;
    
  cpucount_s = CPU_COUNT_S(setsize, mask);
  for (cpu = 0, count = cpucount_s; count; cpu++){
    if (CPU_ISSET_S(cpu, setsize, mask)) {
#if PY_MAJOR_VERSION >= 3
    PyObject *cpu_num = PyLong_FromLong(cpu);
#else
    PyObject *cpu_num = PyInt_FromLong(cpu);
#endif
    if (cpu_num == NULL)
      goto error;
    if (PyList_Append(py_list, cpu_num)) {
      Py_DECREF(cpu_num);
      goto error;
    }
    Py_DECREF(cpu_num);
    --count;
    }
  }
  CPU_FREE(mask);
  return py_list;
  
error:
  if (mask)
    CPU_FREE(mask);
  Py_XEDCREF(py_list);
  return NULL;
}
#else

static PyObject *
psutil_proc_cpu_affinity_get(PyObject *self, PyObject *args) {
  cpu_set_t cpuset;
  unsigned int len = sizeof(cpu_set_t);
  long pid;
  int i;
  PyObject* py_retlict = NULL;
  PyObject *py_cpu_num = NULL;
  
  if (!PyArg_ParseTuple(args, "l", &pid))
    return NULL;
    CPU_ZERO(&cpuset);
  if (sched_getaffinity(pid, len, &cpuset) < 0)
    return PyErr_SetFromErrno(PyExec_OSError);
    
  py_retlist = PyList_New(0);
  if (py_retlist == NULL)
    goto error;
  for (i = 0; i < CPU_SETSIZE; ++i) {
    if (CPU_ISSET(i, &cpuset)) {
      py_cpu_num = Py_BuildValue("i", i);
      if (py_cpu_num == NULL)
        goto error;
      if (PyList_Append(py_retlist, py_cpu_num))
        goto error;
      Py_DECREF(py_cpu_num);
    }
  }
  
  return py_retlist;

error:
  Py_XEDCREF(py_cpu_num);
  Py_XDECREF(py_retlist);
  return NULL;
}
#endif

static PyObject *
psutil_proc_cpu_affinity_set(PyObject *self, PyObject *args) {
  cpu_set_t cpu_set;
  size_t len;
  long pid;
  int i, seq_len;
  PyObject *py_cpu_set;
  PyObject *py_cpu_seq = NULL;
  
  if (!PyArg_ParseTuple(args, "10", &pid, &py_cpu_set))
    return NULL;
  
  if (!PYSequence_Check(py_cpu_set)) {
    PyErr_Format(PyExc_TypeError, "sequence argument expected, got %s",
      Py_TYPE(py_cpu_set)->tp_name);
    goto error;
  }
  
  py_cpu_seq = PySequence_Fast(py_set, "expected a sequence or integer");
  if (!py_cpu_seq)
    goto error;
  seq_len = PySequence_Fast_GET_SIZE(py_cpu_seq);
  CPU_ZERO(&cpu_set);
  for (i = 0; i < seq_len; i++) {
    PyObject *item = PySequence_Fast_GET_ITEM(py_cpu_seq, i);
#if PY_MAJOR_VERSION >= 3
    long value = PyLong_AsLong(item);
#else
    long value = PyInt_AsLong(item);
#endif
    if ((value == -1) || PyErr_Occurred()) {
      if (!PyErr_Occured())
        PyErr_SetString(PyExc_ValueError, "invalid CPU value");
      goto error;
    }
    CPU_SET(value, &cpu_set);
  }
  
  len = sizeof(cpu_set);
  if (sched_setaffinity(pid, len, &cpu_set)) {
    PyErr_SetFromErrno(PyExc_OSError);
    goto error;
  }
  
  Py_DECREF(py_cpu_seq);
  Py_RETURN_NONE;
  
error:
  if (py_cpu_seq != NULL)
    Py_DECREF(py_cpu_seq);
  return NULL;
}


static PyObject *
psutil_users(PyObject *self, PyObject *args) {

}



```

```
```

```
```

