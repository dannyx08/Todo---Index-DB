
<script lang="ts">
    import { onMount } from 'svelte';

    type Todo = {
        id: number;
        text: string;
        completed: boolean;
    };

    let todoText = $state('');
    let todos = $state<Todo[]>([]);

    // 1. Open or create the database
    function openDB(): Promise<IDBDatabase> {
        return new Promise((resolve, reject) => {
            const request = indexedDB.open('TodoDatabase', 1);

            request.onupgradeneeded = () => {
                const db = request.result;

                if (!db.objectStoreNames.contains('todos')) {
                    db.createObjectStore('todos', {
                        keyPath: 'id'
                    });
                }
            };

            request.onsuccess = () => resolve(request.result);
            request.onerror = () => reject(request.error);
        });
    }

    // 2. Wait for a database transaction to finish
    function transactionDone(tx: IDBTransaction): Promise<void> {
        return new Promise((resolve, reject) => {
            tx.oncomplete = () => resolve();
            tx.onerror = () => reject(tx.error);
            tx.onabort = () => reject(tx.error);
        });
    }

    // 3. Retrieve all saved todos
    async function getTodos() {
        const db = await openDB();

        const tx = db.transaction('todos', 'readonly');
        const done = transactionDone(tx);

        const request = tx.objectStore('todos').getAll();

        const result = await new Promise<Todo[]>((resolve, reject) => {
            request.onsuccess = () => resolve(request.result);
            request.onerror = () => reject(request.error);
        });

        await done;
        todos = result;
    }

    // 4. Add a new todo
    async function addTodo() {
        if (!todoText.trim()) return;

        const newTodo: Todo = {
            id: Date.now(),
            text: todoText.trim(),
            completed: false
        };

        const db = await openDB();
        const tx = db.transaction('todos', 'readwrite');

        const done = transactionDone(tx);

        tx.objectStore('todos').add(newTodo);

        await done;

        todos.push(newTodo);
        todoText = '';
    }

    // 5. Delete an individual todo
    async function deleteTodo(id: number) {
        const db = await openDB();

        const tx = db.transaction('todos', 'readwrite');
        const done = transactionDone(tx);

        tx.objectStore('todos').delete(id);

        await done;

        todos = todos.filter(todo => todo.id !== id);
    }

    // 6. Mark todo as completed or incomplete
    async function toggleTodo(id: number) {
        const todo = todos.find(todo => todo.id === id);

        if (!todo) return;

        const updatedTodo = {
            ...todo,
            completed: !todo.completed
        };

        const db = await openDB();

        const tx = db.transaction('todos', 'readwrite');
        const done = transactionDone(tx);

        tx.objectStore('todos').put(updatedTodo);

        await done;

        todos = todos.map(todo =>
            todo.id === id ? updatedTodo : todo
        );
    }

    // 7. Delete all todos
    async function clearTodos() {
        const db = await openDB();

        const tx = db.transaction('todos', 'readwrite');
        const done = transactionDone(tx);

        tx.objectStore('todos').clear();

        await done;

        todos = [];
    }

    // 8. Load saved todos when the page opens
    onMount(() => {
        getTodos();
    });
</script>

<div class="container">
    <h1>TODO LIST</h1>

    <input
        type="text"
        name="name"
        placeholder="Add todo..."
        bind:value={todoText}
        onkeydown={(event) => event.key === 'Enter' && addTodo()}
    >

    <button onclick={addTodo}>Submit</button>
    <button onclick={clearTodos}>Clear All</button>

    {#each todos as todo (todo.id)}
        <div class="todo">
            <input class="checkbox"
                type="checkbox"
                checked={todo.completed}
                onchange={() => toggleTodo(todo.id)}
            >

            <span class:completed={todo.completed}>
                {todo.text}
            </span>

            <button class="cancel" onclick={() => deleteTodo(todo.id)}>X</button>
        </div>
    {/each}
</div>

<style>
.container{
    display: flex;
    flex-direction: column;
    align-items: center;
}

input{
    width: 400px;
    height: 40px;
    border-radius: 5px;
    margin: 5px;
}

button{
    width: 400px;
    height: 40px;
    border-radius: 5px;
    margin: 5px;
}

.cancel{
    background: 0;
    border: 0;
}

.checkbox{
    width: 20px;
    height: 20px;
}

.completed {
    text-decoration: line-through;
}

.todo {
    display: flex;
    align-items: center;
}
</style>
