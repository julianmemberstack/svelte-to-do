<script lang="ts">
  import { onMount, onDestroy } from 'svelte';
  import { writable } from 'svelte/store';
  import { flip } from 'svelte/animate';
  import { fade, scale } from 'svelte/transition';
  import { quintOut } from 'svelte/easing';
  import confetti from 'canvas-confetti';
  import { createEventDispatcher } from 'svelte';
  import { formatDuration } from './utils';

  type Task = {
    id: number;
    title: string;
    description: string;
    dueDate: string;
    status: string;
    timeTracked: number;
  };

  type Category = {
    id: string;
    name: string;
  };

  const tasks = writable<Task[]>([]);
  const categories = writable<Category[]>([
    { id: 'todo', name: 'To Do' },
    { id: 'doing', name: 'Doing' },
    { id: 'done', name: 'Done' }
  ]);
  
  let editingTask: Task | null = null;
  let isModalOpen = false;
  let isCategoryModalOpen = false;
  let newCategoryName = '';
  let editingCategory: Category | null = null;

  let draggedTask: Task | null = null;
  let draggedElement: HTMLElement | null = null;

  let isNewTaskModalOpen = false;
  let newTaskTitle = '';
  let newTaskDescription = '';
  let newTaskDueDate = '';
  let newTaskCategory = '';

  let isTracking = false;
  let trackingStartTime: number | null = null;
  let trackingTaskId: number | null = null;
  let currentTrackedTime = 0;
  let trackingInterval: number | null = null;

  onMount(() => {
    const storedTasks = localStorage.getItem('tasks');
    const storedCategories = localStorage.getItem('categories');
    if (storedTasks) {
      tasks.set(JSON.parse(storedTasks));
    }
    if (storedCategories) {
      categories.set(JSON.parse(storedCategories));
    }

    tasks.subscribe(value => {
      localStorage.setItem('tasks', JSON.stringify(value));
    });

    categories.subscribe(value => {
      localStorage.setItem('categories', JSON.stringify(value));
    });
  });

  function openNewTaskModal() {
    isNewTaskModalOpen = true;
    newTaskTitle = '';
    newTaskDescription = '';
    newTaskDueDate = '';
    newTaskCategory = $categories[0]?.id || '';
  }

  function closeNewTaskModal() {
    isNewTaskModalOpen = false;
  }

  function addTask() {
    if (newTaskTitle.trim()) {
      tasks.update(t => [...t, { 
        id: Date.now(), 
        title: newTaskTitle.trim(), 
        description: newTaskDescription.trim(),
        dueDate: newTaskDueDate,
        status: newTaskCategory,
        timeTracked: 0
      }]);
      closeNewTaskModal();
    }
  }

  function handleDragStart(event: DragEvent, task: Task) {
    draggedTask = task;
    draggedElement = event.target as HTMLElement;
    if (draggedElement) {
      draggedElement.style.opacity = '0.4';
    }
    event.dataTransfer?.setData('text/plain', JSON.stringify(task));
  }

  function handleDragEnd(event: DragEvent) {
    if (draggedElement) {
      draggedElement.style.opacity = '1';
    }
    draggedTask = null;
    draggedElement = null;
  }

  function handleDragOver(event: DragEvent) {
    event.preventDefault();
  }

  function handleDrop(event: DragEvent, newStatus: string) {
    event.preventDefault();
    const taskData = event.dataTransfer?.getData('text/plain');
    if (taskData && draggedTask) {
      const oldStatus = draggedTask.status;
      tasks.update(allTasks => {
        const updatedTasks = allTasks.filter(t => t.id !== draggedTask!.id);
        const insertIndex = updatedTasks.filter(t => t.status === newStatus).length;
        updatedTasks.splice(
          updatedTasks.findIndex(t => t.status === newStatus) + insertIndex,
          0,
          { ...draggedTask!, status: newStatus }
        );
        return updatedTasks;
      });

      // Trigger confetti when a task is moved to 'done'
      if (oldStatus !== 'done' && newStatus === 'done') {
        const rect = (event.currentTarget as HTMLElement).getBoundingClientRect();
        const x = (event.clientX - rect.left) / rect.width;
        const y = (event.clientY - rect.top) / rect.height;
        triggerConfetti(x, y);
      }
    }
  }

  function triggerConfetti(x: number, y: number) {
    confetti({
      particleCount: 100,
      spread: 70,
      origin: { x, y }
    });
  }

  function openEditModal(task: Task) {
    editingTask = { ...task };
    isModalOpen = true;
  }

  function closeModal() {
    if (isTracking) {
      stopTracking();
    }
    isModalOpen = false;
    editingTask = null;
  }

  function saveTask() {
    if (editingTask) {
      tasks.update(t => t.map(task => task.id === editingTask!.id ? editingTask! : task));
      closeModal();
    }
  }

  function deleteTask() {
    if (editingTask) {
      tasks.update(t => t.filter(task => task.id !== editingTask!.id));
      closeModal();
    }
  }

  function formatDate(dateString: string) {
    if (!dateString) return '';
    const date = new Date(dateString);
    return date.toLocaleDateString('en-US', { month: 'short', day: 'numeric' });
  }

  function truncateDescription(description: string, maxLength: number = 50): string {
    if (description.length <= maxLength) return description;
    return description.slice(0, maxLength).trim() + '...';
  }

  function openCategoryModal() {
    isCategoryModalOpen = true;
  }

  function closeCategoryModal() {
    isCategoryModalOpen = false;
    editingCategory = null;
    newCategoryName = '';
  }

  function addCategory() {
    if (newCategoryName.trim()) {
      categories.update(c => [...c, { id: Date.now().toString(), name: newCategoryName.trim() }]);
      newCategoryName = '';
    }
  }

  function startEditingCategory(category: Category) {
    editingCategory = { ...category };
  }

  function saveCategory() {
    if (editingCategory) {
      categories.update(c => c.map(cat => cat.id === editingCategory!.id ? editingCategory! : cat));
      editingCategory = null;
    }
  }

  function deleteCategory(categoryId: string) {
    categories.update(c => c.filter(cat => cat.id !== categoryId));
    tasks.update(t => t.filter(task => task.status !== categoryId));
  }

  function moveCategory(categoryId: string, direction: 'up' | 'down') {
    categories.update(c => {
      const index = c.findIndex(cat => cat.id === categoryId);
      if ((direction === 'up' && index > 0) || (direction === 'down' && index < c.length - 1)) {
        const newIndex = direction === 'up' ? index - 1 : index + 1;
        const updatedCategories = [...c];
        [updatedCategories[index], updatedCategories[newIndex]] = [updatedCategories[newIndex], updatedCategories[index]];
        return updatedCategories;
      }
      return c;
    });
  }

  function startTracking(taskId: number) {
    if (isTracking && trackingTaskId === taskId) {
      stopTracking();
    } else {
      if (isTracking) {
        stopTracking();
      }
      isTracking = true;
      trackingStartTime = Date.now();
      trackingTaskId = taskId;
      currentTrackedTime = editingTask.timeTracked || 0;
      
      trackingInterval = setInterval(() => {
        if (trackingStartTime) {
          const elapsedTime = Math.floor((Date.now() - trackingStartTime) / 1000);
          currentTrackedTime = (editingTask.timeTracked || 0) + elapsedTime;
        }
      }, 1000);
    }
  }

  function stopTracking() {
    if (isTracking && trackingStartTime && trackingTaskId) {
      const elapsedTime = Math.floor((Date.now() - trackingStartTime) / 1000);
      tasks.update(allTasks => 
        allTasks.map(task => 
          task.id === trackingTaskId 
            ? { ...task, timeTracked: (task.timeTracked || 0) + elapsedTime }
            : task
        )
      );
      if (editingTask && editingTask.id === trackingTaskId) {
        editingTask.timeTracked = (editingTask.timeTracked || 0) + elapsedTime;
        currentTrackedTime = editingTask.timeTracked;
      }
      isTracking = false;
      trackingStartTime = null;
      trackingTaskId = null;
      if (trackingInterval) {
        clearInterval(trackingInterval);
        trackingInterval = null;
      }
    }
  }

  onDestroy(() => {
    if (trackingInterval) {
      clearInterval(trackingInterval);
    }
  });
</script>

<main class="max-w-full mx-auto p-4 sm:p-6 md:p-10">
  <div class="mb-6 sm:mb-8 flex flex-col sm:flex-row justify-between items-start sm:items-center gap-4 sm:gap-0">
    <h2 class="text-xl sm:text-2xl font-semibold text-black">Categories</h2>
    <div class="flex flex-wrap gap-2 sm:gap-4">
      <button
        on:click={openNewTaskModal}
        class="bg-green-600 text-white px-3 py-1 sm:px-4 sm:py-2 text-sm sm:text-base rounded-lg hover:bg-green-700 transition duration-300 shadow-sm"
      >
        Add Task
      </button>
      <button
        on:click={openCategoryModal}
        class="bg-blue-700 text-white px-3 py-1 sm:px-4 sm:py-2 text-sm sm:text-base rounded-lg hover:bg-blue-800 transition duration-300 shadow-sm"
      >
        Edit Categories
      </button>
    </div>
  </div>

  <div class="flex flex-col sm:flex-row gap-4 sm:gap-6 sm:overflow-x-auto pb-6">
    {#each $categories as category (category.id)}
      <div class="w-full sm:w-80 flex-shrink-0">
        <div class="bg-gray-50 rounded-lg p-4 h-full shadow-sm">
          <h2 class="text-lg sm:text-xl font-semibold text-black mb-4">{category.name}</h2>
          <div
            class="min-h-[200px]"
            on:dragover={handleDragOver}
            on:drop={(e) => handleDrop(e, category.id)}
          >
            {#each $tasks.filter(t => t.status === category.id) as task (task.id)}
              <div animate:flip={{ duration: 300 }}>
                <div
                  class="bg-white rounded-lg p-3 mb-3 cursor-pointer hover:shadow-md transition duration-300 border border-gray-100 shadow-sm"
                  draggable="true"
                  on:dragstart={(e) => handleDragStart(e, task)}
                  on:dragend={handleDragEnd}
                  on:click={() => openEditModal(task)}
                >
                  <p class="font-medium mb-2 text-gray-800">{task.title}</p>
                  {#if task.description}
                    <p class="text-sm text-gray-600 mb-2">{truncateDescription(task.description)}</p>
                  {/if}
                  <div class="flex justify-between items-center">
                    {#if task.dueDate}
                      <span class="text-xs bg-blue-50 text-blue-600 px-2 py-1 rounded-full">{formatDate(task.dueDate)}</span>
                    {/if}
                    {#if task.timeTracked}
                      <span class="text-xs bg-green-50 text-green-600 px-2 py-1 rounded-full">{formatDuration(task.timeTracked)}</span>
                    {/if}
                  </div>
                </div>
              </div>
            {/each}
          </div>
        </div>
      </div>
    {/each}
  </div>

  <!-- New Task Modal -->
  {#if isNewTaskModalOpen}
    <div 
      class="fixed inset-0 bg-black bg-opacity-50 flex items-center justify-center z-50 p-4" 
      on:click={closeNewTaskModal}
      transition:fade={{ duration: 200 }}
    >
      <div 
        class="bg-white rounded-xl p-4 sm:p-8 w-full max-w-md shadow-md" 
        on:click|stopPropagation
        transition:scale={{ duration: 200, start: 0.95, opacity: 1, easing: quintOut }}
      >
        <h2 class="text-xl sm:text-2xl font-bold mb-4 sm:mb-6 text-black">Add New Task</h2>
        <form on:submit|preventDefault={addTask} class="space-y-4">
          <input
            bind:value={newTaskTitle}
            placeholder="Task title"
            class="w-full p-2 sm:p-3 border border-gray-200 rounded-lg focus:outline-none focus:ring-2 focus:ring-blue-400 focus:border-transparent transition duration-300"
          />
          <textarea
            bind:value={newTaskDescription}
            placeholder="Task description"
            class="w-full p-2 sm:p-3 border border-gray-200 rounded-lg h-24 sm:h-32 resize-none focus:outline-none focus:ring-2 focus:ring-blue-400 focus:border-transparent transition duration-300"
          ></textarea>
          <input
            type="date"
            bind:value={newTaskDueDate}
            class="w-full p-2 sm:p-3 border border-gray-200 rounded-lg focus:outline-none focus:ring-2 focus:ring-blue-400 focus:border-transparent transition duration-300"
          />
          <select
            bind:value={newTaskCategory}
            class="w-full p-2 sm:p-3 border border-gray-200 rounded-lg focus:outline-none focus:ring-2 focus:ring-blue-400 focus:border-transparent transition duration-300"
          >
            {#each $categories as category}
              <option value={category.id}>{category.name}</option>
            {/each}
          </select>
          <div class="flex justify-end gap-2 sm:gap-4">
            <button
              type="button"
              on:click={closeNewTaskModal}
              class="bg-gray-100 text-gray-700 px-4 sm:px-6 py-2 rounded-lg hover:bg-gray-200 transition duration-300 shadow-sm"
            >Cancel</button>
            <button
              type="submit"
              class="bg-blue-700 text-white px-4 sm:px-6 py-2 rounded-lg hover:bg-blue-800 transition duration-300 shadow-sm"
            >Add Task</button>
          </div>
        </form>
      </div>
    </div>
  {/if}

  <!-- Existing task edit modal -->
  {#if isModalOpen}
    <div 
      class="fixed inset-0 bg-black bg-opacity-50 flex items-center justify-center z-50 p-4" 
      on:click={closeModal}
      transition:fade={{ duration: 200 }}
    >
      <div 
        class="bg-white rounded-xl p-4 sm:p-8 w-full max-w-md shadow-md" 
        on:click|stopPropagation
        transition:scale={{ duration: 200, start: 0.95, opacity: 1, easing: quintOut }}
      >
        <h2 class="text-xl sm:text-2xl font-bold mb-4 sm:mb-6 text-black">Edit Task</h2>
        <input
          bind:value={editingTask.title}
          placeholder="Task title"
          class="w-full mb-4 p-2 sm:p-3 border border-gray-200 rounded-lg focus:outline-none focus:ring-2 focus:ring-blue-400 focus:border-transparent transition duration-300"
        />
        <textarea
          bind:value={editingTask.description}
          placeholder="Task description"
          class="w-full mb-4 p-2 sm:p-3 border border-gray-200 rounded-lg h-24 sm:h-32 resize-none focus:outline-none focus:ring-2 focus:ring-blue-400 focus:border-transparent transition duration-300"
        ></textarea>
        <input
          type="date"
          bind:value={editingTask.dueDate}
          class="w-full mb-4 p-2 sm:p-3 border border-gray-200 rounded-lg focus:outline-none focus:ring-2 focus:ring-blue-400 focus:border-transparent transition duration-300"
        />
        <select
          bind:value={editingTask.status}
          class="w-full mb-4 p-2 sm:p-3 border border-gray-200 rounded-lg focus:outline-none focus:ring-2 focus:ring-blue-400 focus:border-transparent transition duration-300"
        >
          {#each $categories as category}
            <option value={category.id}>{category.name}</option>
          {/each}
        </select>

        <!-- Time tracking section -->
        <div class="bg-gray-50 rounded-lg p-4 mb-6 border border-gray-200">
          <h3 class="text-lg font-semibold mb-2 text-gray-800">Time Tracking</h3>
          <div class="flex items-center justify-between">
            <p class="text-gray-600">
              Total time: <span class="font-medium text-gray-800">{formatDuration(isTracking && trackingTaskId === editingTask.id ? currentTrackedTime : editingTask.timeTracked || 0)}</span>
            </p>
            <button
              on:click={() => startTracking(editingTask.id)}
              class={`px-4 py-2 rounded-lg transition duration-300 shadow-sm text-white ${
                isTracking && trackingTaskId === editingTask.id 
                  ? 'bg-red-500 hover:bg-red-600' 
                  : 'bg-green-500 hover:bg-green-600'
              }`}
            >
              {isTracking && trackingTaskId === editingTask.id ? 'Stop' : 'Start'}
            </button>
          </div>
        </div>

        <div class="flex justify-between items-center">
          <button
            on:click={deleteTask}
            class="bg-red-500 text-white px-3 sm:px-4 py-2 rounded-lg hover:bg-red-600 transition duration-300 shadow-sm"
          >Delete</button>
          <div class="flex gap-2 sm:gap-4">
            <button
              on:click={saveTask}
              class="bg-blue-700 text-white px-4 sm:px-6 py-2 rounded-lg hover:bg-blue-800 transition duration-300 shadow-sm"
            >Save</button>
            <button
              on:click={closeModal}
              class="bg-gray-100 text-gray-700 px-4 sm:px-6 py-2 rounded-lg hover:bg-gray-200 transition duration-300 shadow-sm"
            >Cancel</button>
          </div>
        </div>
      </div>
    </div>
  {/if}

  <!-- Category management modal -->
  {#if isCategoryModalOpen}
    <div 
      class="fixed inset-0 bg-black bg-opacity-50 flex items-center justify-center z-50 p-4" 
      on:click={closeCategoryModal}
      transition:fade={{ duration: 200 }}
    >
      <div 
        class="bg-white rounded-xl p-4 sm:p-8 w-full max-w-md shadow-md" 
        on:click|stopPropagation
        transition:scale={{ duration: 200, start: 0.95, opacity: 1, easing: quintOut }}
      >
        <h2 class="text-xl sm:text-2xl font-bold mb-4 sm:mb-6 text-black">Manage Categories</h2>
        
        <!-- Add new category form -->
        <form on:submit|preventDefault={addCategory} class="mb-4 sm:mb-6 flex gap-2 sm:gap-3">
          <input
            bind:value={newCategoryName}
            placeholder="New category name"
            class="flex-grow py-2 px-3 rounded-lg bg-white border border-gray-200 focus:outline-none focus:ring-2 focus:ring-blue-400 focus:border-transparent transition duration-300"
          />
          <button type="submit" class="bg-blue-700 text-white px-3 sm:px-4 py-2 rounded-lg hover:bg-blue-800 transition duration-300 shadow-sm">
            Add
          </button>
        </form>

        <!-- List of existing categories -->
        <div class="space-y-3 sm:space-y-4 max-h-60 overflow-y-auto">
          {#each $categories as category (category.id)}
            <div class="flex items-center justify-between">
              {#if editingCategory && editingCategory.id === category.id}
                <input
                  bind:value={editingCategory.name}
                  class="flex-grow py-1 px-2 border-b border-blue-300 focus:outline-none focus:border-blue-500"
                  on:blur={saveCategory}
                  on:keypress={(e) => e.key === 'Enter' && saveCategory()}
                />
              {:else}
                <span>{category.name}</span>
              {/if}
              <div class="flex gap-1 sm:gap-2">
                <button on:click={() => moveCategory(category.id, 'up')} class="text-gray-500 hover:text-gray-700">↑</button>
                <button on:click={() => moveCategory(category.id, 'down')} class="text-gray-500 hover:text-gray-700">↓</button>
                <button on:click={() => startEditingCategory(category)} class="text-blue-500 hover:text-blue-700">Edit</button>
                <button on:click={() => deleteCategory(category.id)} class="text-red-500 hover:text-red-700">Delete</button>
              </div>
            </div>
          {/each}
        </div>

        <button
          on:click={closeCategoryModal}
          class="mt-4 sm:mt-6 bg-gray-100 text-gray-700 px-4 sm:px-6 py-2 rounded-lg hover:bg-gray-200 transition duration-300 shadow-sm"
        >Close</button>
      </div>
    </div>
  {/if}
</main>

<style>
  :global(body) {
    background-color: #f0f4f8;
  }
</style>
