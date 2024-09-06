<template>
    <div>
      <div class="mb-4 flex justify-between items-center">
        <h2 class="text-2xl font-semibold">Mis tareas</h2>
        <Button label="Agregar nueva tarea" @click="showAddDialog = true" />
      </div>
  
      <DataTable :value="tasks" class="p-datatable-sm" :loading="loading">
        <Column field="title" header="Titulo"></Column>
        <Column field="priority" header="Prioridad"></Column>
        <Column field="dueDate" header="Fecha creación">
            <template #body="slotProps">
            {{ formatDate(slotProps.data.dueDate) }}
        </template>
        </Column>
        <Column field="status" header="Estado">
          <template #body="slotProps">
            <Tag :value="slotProps.data.status" :severity="getStatusSeverity(slotProps.data.status)">
              {{ slotProps.data.status }}
            </Tag>
          </template>
        </Column>
        <Column header="Acciones">
          <template #body="slotProps">
            <Button icon="pi pi-pencil" class="p-button-text" @click="editTask(slotProps.data)" />
          </template>
        </Column>
      </DataTable>
  
      <Dialog v-model:visible="showAddDialog" :header="dialogMode === 'Agregar' ? 'Agregar tarea' : 'Editar tarea'" :style="{width: '50vw'}">
        <div class="p-fluid">
          <div class="p-field">
            <label for="title">Titulo</label>
            <InputText id="title" v-model="newTask.title" />
          </div>
          <div class="p-field">
            <label for="description">Descripción</label>
            <InputText id="description" v-model="newTask.description" />
            </div>
          
          <div class="p-field">
            <label for="priority">Prioridad</label>
            <Dropdown id="priority" v-model="newTask.priority" :options="priorities" optionLabel="label" optionValue="value" placeholder="Select Priority" />
          </div>
          <div class="p-field">
            <label for="dueDate">Fecha de creación</label>
            <Calendar id="dueDate" v-model="newTask.dueDate" dateFormat="dd/mm/yy" />
          </div>


          <div class="p-field">
            <label for="status">Estado</label>
            <Dropdown id="status" v-model="newTask.status" :options="statuses" optionLabel="label" optionValue="value" placeholder="Select Status" />
          </div>
        </div>
        <template #footer>
          <Button label="Cancelar" icon="pi pi-times" @click="closeDialog" class="p-button-text"/>
          <Button :label="dialogMode === 'Agregar' ? 'Actualizar' : 'Guardar'" icon="pi pi-check" @click="dialogMode === 'add' ? addTask() : updateTask()" />
        </template>
      </Dialog>
    </div>
  </template>
  
  <script setup>
  import { ref, onMounted } from 'vue'
  import Tag from 'primevue/tag'
  import Dropdown from 'primevue/dropdown'
  import Calendar from 'primevue/calendar'
  
  const formatDate = (date) => {
  if (!date) return ''
  const formattedDate = new Date(date).toLocaleDateString('es-ES', {
    day: '2-digit',
    month: '2-digit',
    year: 'numeric'
  })
  return formattedDate
}

  const tasks = ref([])
  const loading = ref(true)
  const showAddDialog = ref(false)
  const dialogMode = ref('add')
  const newTask = ref({
    title: '',
    description: '',
    priority: '',
    dueDate: formatDate(new Date()),
    status: 'Pendiente',
    isCompleted: false
})

  
  const priorities = ref([
    { label: 'High', value: 'High' },
    { label: 'Medium', value: 'Medium' },
    { label: 'Low', value: 'Low' }
  ])
  
  const statuses = ref([
    { label: 'En progreso', value: 'En progreso' },
    { label: 'Terminado', value: 'Terminado' },
    { label: 'Cancelado', value: 'Cancelado' },
    { label: 'Pendiente', value: 'Pendiente' }
  ])
  
  onMounted(async () => {
    await fetchTasks()
  })
  
  const fetchTasks = async () => {
    loading.value = true
    try {

      const response = await fetch('http://localhost:5000/tasks/')
      const data = await response.json()
      tasks.value = data.map(task => ({
        ...task,
        priority: 'Medium',  // Valor predeterminado para priority
        dueDate: new Date(),  // Valor predeterminado para dueDate
        status: task.isCompleted ? 'Terminado' : 'En progreso'  // Mapeo del status
      }))
    } catch (error) {
      console.error('Error fetching tasks:', error)
    } finally {
      loading.value = false
    }
  }
  
  const addTask = async () => {
    try {
        if (!newTask.value.dueDate) {
        newTask.value.dueDate = new Date() // If no due date, set to current date
    }

    const formattedDueDate = formatDate(newTask.value.dueDate);

      const response = await fetch('http://localhost:5000/tasks/', {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
        },
        body: JSON.stringify(newTask.value),
      })
      const addedTask = await response.json()
    //   tasks.value.push(addedTask)
    tasks.value.push({
      ...addedTask,
      priority: newTask.value.priority || 'Medium', // Use provided or default value
      dueDate: new Date(newTask.value.dueDate), // Convert to Date object if necessary
      status: newTask.value.isCompleted ? 'Terminado' : 'En progreso', // Map status
    });
      closeDialog()
    } catch (error) {
      console.error('Error adding task:', error)
    }
  }
  
  const updateTask = async () => {
  try {
    // Filtrar solo los campos relevantes para actualizar
    const formattedDueDate = formatDate(newTask.value.dueDate); // 
    const taskData = {
      title: newTask.value.title,
      description: newTask.value.description,
      dueDate: formattedDueDate,
      isCompleted: newTask.value.status === 'Terminado', // Asumiendo que el status 'Terminado' indica que está completada
    };

    // Hacer la solicitud PUT con el ID correcto
    const response = await fetch(`http://localhost:5000/tasks/${newTask.value.id}`, {
      method: 'PUT',
      headers: {
        'Content-Type': 'application/json',
      },
      body: JSON.stringify(taskData), // Enviar los datos como JSON
    });

    if (response.ok) {
      const updatedTask = await response.json();
      const index = tasks.value.findIndex(task => task.id === updatedTask.id);

      // Actualizar la tarea en el array de tareas
      if (index !== -1) {
        tasks.value[index] = {
          ...updatedTask,
          priority: newTask.value.priority || 'Medium',  // Asegurar prioridad por defecto
          dueDate: new Date(updatedTask.dueDate),  // Convertir la fecha si es necesario
          status: updatedTask.isCompleted ? 'Terminado' : newTask.value.status,  // Actualizar el estado
        };
      }

      closeDialog(); // Cerrar el diálogo
    } else {
      console.error('Error updating task:', await response.text());
    }
  } catch (error) {
    console.error('Error updating task:', error);
  }
};

  
  const editTask = (task) => {
  newTask.value = JSON.parse(JSON.stringify(task)); // Clonar el objeto
  dialogMode.value = 'edit';
  showAddDialog.value = true;
};

  
  const closeDialog = () => {
    showAddDialog.value = false
    newTask.value = { title: '', priority: '', dueDate: '', status: '' }
    dialogMode.value = 'add'
  }
  
  const getStatusSeverity = (status) => {
    switch (status) {
      case 'En progreso':
        return 'warning';
      case 'Terminado':
        return 'success';
      case 'Cancelado':
        return 'danger';
      case 'Pendiente':
        return 'info';
      default:
        return null;
    }
  }
  </script>
  