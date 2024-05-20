<script>
    import { db } from "../../lib/firebase/firebase";
    import { authHandlers, authStore } from "../../store/store";
    import { getDoc, doc, setDoc } from "firebase/firestore";
    import TodoItem from "../../components/TodoItem.svelte";

    let todoList = [];
    let clockInTimes = [];
    let clockOutTimes = [];
    let timeMapArray = [];
    let currTodo = "";
    let error = false;

    authStore.subscribe((curr) => {
        //todoList = curr.data.todos;
        clockInTimes = curr.data.clockIn;
        clockOutTimes = curr.data.clockOut;
        timeMapArray = curr.data.times;
    });

    function reloadPage() {
        location.reload();
    }

    function addTodo() {
        error = false;
        if (!currTodo) {
            error = true;
        }
        todoList = [...todoList, [currTodo, "EMPTY"]];
        currTodo = "";
    }

    function clockIn() {
        error = false;
        if (!currTodo) {
            error = true;
        }
        todoList = [...todoList, [new Date()]];
        clockInTimes = [...clockInTimes, new Date()];
        clockOutTimes = [...clockOutTimes, "CURRENTLY CLOCKED IN"];
        currTodo = "";

    
        if (timeMapArray.length != 0 && timeMapArray[timeMapArray.length - 1].clockOut == "CURRENTLY CLOCKED IN") {
            alert("Unable to clock in, you are already clocked in!");
            return;
        }

        let currentTimes = {
            clockIn: new Date(),
            clockOut: "CURRENTLY CLOCKED IN",
            totalTime: null,
        }
        timeMapArray = [...timeMapArray, currentTimes]
        saveTodos();
        console.log(timeMapArray);
        

        
    }

    function clockOut() {
        error = false;
        if (!currTodo) {
            error = true;
        }

        if (timeMapArray.length != 0 && timeMapArray[timeMapArray.length - 1].clockOut == "CURRENTLY CLOCKED IN") {
            timeMapArray[timeMapArray.length - 1].clockOut = new Date();
            timeMapArray[timeMapArray.length - 1].totalTime = (Math.abs(timeMapArray[timeMapArray.length - 1].clockOut - timeMapArray[timeMapArray.length - 1].clockIn.toDate()) / (1000 * 60 * 60));
            saveTodos();
        } else {
            alert("Unable to clock out, you are not clocked in!");
        }

        saveTodos();

        currTodo = "";
    }

    function editTodo(index) {
        let newTodoList = [...todoList].filter((val, i) => {
            console.log(i, index, i !== index);
            return i != index;
        });
        currTodo = todoList[index];
        todoList = newTodoList;
    }

    function removeTodo(index) {
        let newTodoList = [...todoList].filter((val, i) => {
            console.log(i, index, i !== index);
            return i != index;
        });
        todoList = newTodoList;
    }

    async function saveTodos() {
        try {
            const userRef = doc(db, "users", $authStore.user.uid);
            await setDoc(
                userRef,
                {
                    //todos: todoList,
                    clockIn: clockInTimes,
                    clockOut: clockOutTimes,
                    times: timeMapArray,
                },
                { merge: true }
            );
        } catch (err) {
            console.log("There was an error saving your information");
        }

        reloadPage();
    }
</script>

{#if !$authStore.loading}
    <div class="mainContainer">
        <div class="headerContainer">
            <h1>FTP Dispatch Time Tracking</h1>
            <div class="headerBtns">
                
                <button on:click={authHandlers.logout}>
                    <i class="fa-solid fa-right-from-bracket" />
                    <p>Logout</p></button
                >
            </div>
        </div>
        <main>
            {#if clockInTimes.length === 0}
                <p>You Currently Have no History!</p>
            {/if}
            {#each timeMapArray as clockInItem, index}
            <div class="todo">
                <p>
                    {index + 1}. {timeMapArray[index].clockIn.toDate().toLocaleDateString('en-US')} {timeMapArray[index].clockIn.toDate().toLocaleTimeString('en-US')} - {timeMapArray[index].clockOut == "CURRENTLY CLOCKED IN" ? "CURRENTLY CLOCKED IN" : timeMapArray[index].clockOut.toDate().toLocaleDateString('en-us')} {timeMapArray[index].clockOut == "CURRENTLY CLOCKED IN" ? "" : timeMapArray[index].clockOut.toDate().toLocaleTimeString('en-us')}
                </p>
            </div>
            {/each}
        </main>
        <div class={"enterTodo " + (error ? "errorBorder" : "")}>
            
            <button on:click={clockIn}>Clock In</button>
            <button on:click={clockOut}>Clock Out</button>
        </div>
    </div>
{/if}

<style>
    .todo {
        border-left: 1px solid cyan;
        padding: 8px 14px;
        display: flex;
        align-items: center;
        justify-content: space-between;
    }

    .actions {
        display: flex;
        align-items: center;
        gap: 14px;
        font-size: 1.3rem;
    }

    .actions i {
        cursor: pointer;
    }

    .actions i:hover {
        color: coral;
    }
    .mainContainer {
        display: flex;
        flex-direction: column;
        min-height: 100vh;
        gap: 24px;
        padding: 24px;
        width: 100%;
        max-width: 1000px;
        margin: 0 auto;
    }

    .headerContainer {
        display: flex;
        align-items: center;
        justify-content: space-between;
    }

    .headerBtns {
        display: flex;
        align-items: center;
        gap: 14px;
    }

    .headerContainer button {
        background: #003c5b;
        color: white;
        padding: 10px 18px;
        border: none;
        border-radius: 4px;
        font-weight: 700;
        display: flex;
        align-items: center;
        gap: 10px;
        cursor: pointer;
    }

    .headerContainer button i {
        font-size: 1.1rem;
    }

    .headerContainer button:hover {
        opacity: 0.7;
    }

    main {
        display: flex;
        flex-direction: column;
        gap: 8px;
        flex: 1;
    }

    .enterTodo {
        display: flex;
        width: 250px;
        align-items: stretch;
        border: 1px solid #0891b2;
        border-radius: 10px;
        overflow: hidden;
    }

    .errorBorder {
        border-color: coral !important;
    }

    .enterTodo input {
        background: transparent;
        border: none;
        padding: 14px;
        color: white;
        flex: 1;
    }

    .enterTodo input:focus {
        outline: none;
    }

    .enterTodo button {
        height: 50px;
        width: 125px;
        padding: 0 28px;
        background: #003c5b;
        border: none;
        color: cyan;
        font-weight: 600;
        cursor: pointer;
    }

    .enterTodo button:hover {
        background: transparent;
    }
</style>
