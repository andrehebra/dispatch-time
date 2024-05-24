<script>
    import { db } from "../../lib/firebase/firebase";
    import { authHandlers, authStore } from "../../store/store";
    import { getDoc, doc, setDoc } from "firebase/firestore";
    import { collection, getDocs } from "firebase/firestore";
    import TodoItem from "../../components/TodoItem.svelte";

    let todoList = [];
    let clockInTimes = [];
    let clockOutTimes = [];
    let timeMapArray = [];
    let currTodo = "";
    let error = false;
    let admin = false;
    let adminUsers = [];

    let googleMapsCode = "AIzaSyBfM7u1Q5ZDBqdmQYUOQLYQ-2rLVRlPuLQ";

    /*
    geolocator.config({
        language: "en",
        google: {
            version: "3",
            key: googleMapsCode,
        }
    });
 
    window.onload = function () {
        var options = {
            enableHighAccuracy: true,
            timeout: 5000,
            maximumWait: 10000,     // max wait time for desired accuracy
            maximumAge: 0,          // disable cache
            desiredAccuracy: 30,    // meters
            fallbackToIP: true,     // fallback to IP if Geolocation fails or rejected
            addressLookup: true,    // requires Google API key if true
            timezone: true,         // requires Google API key if true
            map: "map-canvas",      // interactive map element id (or options object)
            staticMap: true         // get a static map image URL (boolean or options object)
        };
        geolocator.locate(options, function (err, location) {
            if (err) return console.log(err);
            console.log(location);
        });
    };
    */
    
    let hasLocation = false;
    let location;

    const successCallback = (position) => {
        console.log(position);
        location = position;
        hasLocation = true;
    };

    const errorCallback = (error) => {
        alert("You must allow location access for this tool to work. You will be unable until you edit your location permissions.")
    };

    navigator.geolocation.getCurrentPosition(successCallback, errorCallback);

    
    authStore.subscribe((curr) => {
        //todoList = curr.data.todos;
        clockInTimes = curr.data.clockIn;
        clockOutTimes = curr.data.clockOut;
        timeMapArray = curr.data.times;
        if (curr.user != null) {
            if (curr.user.uid != null) {
                if (curr.user.uid == "691DpeS3ATWdkFygcrC0cefMxLl1" || curr.user.uid == "PzHTuv5VaAdMQsgp0RFGSlJR9lJ2") {
                    admin = true;
                    getAllUserTimes();
                }
            }
        }
    });

    function reloadPage() {
        location.reload();
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

    async function getAllUserTimes() {
        try {
            const nonAdminUser = collection(db, "users");
            const docSnap = await getDocs(nonAdminUser);
            //const usersData = usersSnapshot.docs.map(doc => ({ id: doc.id, ...doc.data() }));

            const documents = docSnap.docs.map(doc => ({ id: doc.id, ...doc.data() }));
            console.log(documents);

            adminUsers = documents;

            return documents;
            
            //return usersData;
        } catch (error) {
            console.error("Error fetching users data: ", error);
            throw error;
        }
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

    if (admin == true) {
        console.log("hello");
        adminUsers = getAllUserTimes();
        
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
            {#if admin === false}
            <div class={"enterTodo " + (error ? "errorBorder" : "")}>
            
                <button on:click={clockIn}>Clock In</button>
                <button on:click={clockOut}>Clock Out</button>
            </div>
            {#if clockInTimes.length === 0}
                <p>You Currently Have no History!</p>
            {/if}
            {#each timeMapArray as clockInItem, index}
                <div class="todo">
                    <p>
                        {index + 1}. {clockInItem.clockIn.toDate().toLocaleDateString('en-US')} {clockInItem.clockIn.toDate().toLocaleTimeString('en-US')}   -   {clockInItem.clockOut == "CURRENTLY CLOCKED IN" ? "CURRENTLY CLOCKED IN" : clockInItem.clockOut.toDate().toLocaleDateString('en-us')} {clockInItem.clockOut == "CURRENTLY CLOCKED IN" ? "" : clockInItem.clockOut.toDate().toLocaleTimeString('en-us')}
                    </p>
                </div>
            {/each}
            {:else if admin === true}
                {#each adminUsers as user}
                    {#if user.Name != null}
                        <h1>{user.Name}</h1>
                        {#each user.times as clockInItem, index}
                            {#if clockInItem.clockIn.toDate().getHours() > 6 || (clockInItem.clockIn.toDate().getHours() == 6 && clockInItem.clockIn.toDate().getMinutes() > 30)}
                                <div class="todo late">
                                    <p>
                                        {index + 1}. {clockInItem.clockIn.toDate().toLocaleDateString('en-US')} {clockInItem.clockIn.toDate().toLocaleTimeString('en-US')}   -   {clockInItem.clockOut == "CURRENTLY CLOCKED IN" ? "CURRENTLY CLOCKED IN" : clockInItem.clockOut.toDate().toLocaleDateString('en-us')} {clockInItem.clockOut == "CURRENTLY CLOCKED IN" ? "" : clockInItem.clockOut.toDate().toLocaleTimeString('en-us')}
                                    </p>
                                    <button>hello</button>
                                </div>
                            {:else}
                                <div class="todo">
                                    <p>
                                        {index + 1}. {clockInItem.clockIn.toDate().toLocaleDateString('en-US')} {clockInItem.clockIn.toDate().toLocaleTimeString('en-US')}   -   {clockInItem.clockOut == "CURRENTLY CLOCKED IN" ? "CURRENTLY CLOCKED IN" : clockInItem.clockOut.toDate().toLocaleDateString('en-us')} {clockInItem.clockOut == "CURRENTLY CLOCKED IN" ? "" : clockInItem.clockOut.toDate().toLocaleTimeString('en-us')}
                                    </p>
                                </div>
                            {/if}
                            
                        {/each}
                    {/if}
                {/each}
            {/if}
        </main>
        
    </div>
{/if}

<style>
    .late {
        background-color: red;
    }
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
