<script>
    import { db } from "../../lib/firebase/firebase";
    import { authHandlers, authStore } from "../../store/store";
    import { getDoc, doc, setDoc } from "firebase/firestore";
    import { collection, getDocs } from "firebase/firestore";
    import TodoItem from "../../components/TodoItem.svelte";

    import { Select } from 'flowbite-svelte';

    import { DateInput } from 'date-picker-svelte';

    let tempDate = new Date();

    let editTimeModal = false;

    let todoList = [];
    let clockInTimes = [];
    let clockOutTimes = [];
    let timeMapArray = [];
    let currTodo = "";
    let error = false;
    let admin = false;
    let adminUsers = [];
    let adminUsersConverted = [];



    let periodArray = [];
    let currentDate = new Date();
    let periodStartDate = new Date("April 28, 2024 00:00:00");
    let earliestDate = new Date(currentDate);
    earliestDate.setDate(earliestDate.getDate() - 30);
    let nextPeriod = new Date(periodStartDate.getTime());
    
    let tempNextDate = new Date(periodStartDate.getTime() + (13*24*60*60*1000));
    

    
    while (nextPeriod < currentDate) {
        nextPeriod = new Date(nextPeriod.getTime() + (14*24*60*60*1000));
        tempNextDate = new Date(nextPeriod.getTime() + (13*24*60*60*1000));
        if (earliestDate <= nextPeriod) {
            periodArray.push({
                date: nextPeriod,
                ISODate: nextPeriod.toISOString().slice(0, -5),
                value: nextPeriod.toISOString().slice(0, -5),
                name: nextPeriod.toLocaleDateString('en-US') + " - " + tempNextDate.toLocaleDateString('en-US'),
            })
        }
        
        console.log(periodArray);
    }
    let selectedPeriod = nextPeriod;

    let hoursArray = [];
    let totalHours = 0;
    function calculatePayHours() {
        selectedPeriod = new Date(selectedPeriod);

        hoursArray = [];

        let endOfPeriod = new Date(selectedPeriod.getTime() + (14*24*60*60*1000));
        console.log(endOfPeriod);

        for (let i = 0; i < adminUsers.length; i++) {
            if (adminUsers[i].Name != null) {
                totalHours = 0;
                for (let j = 0; j < adminUsers[i].times.length; j++) {
                    if (adminUsers[i].times[j].clockIn.toDate() >= selectedPeriod && adminUsers[i].times[j].clockIn.toDate() <= endOfPeriod) {
                        if (adminUsers[i].times[j].clockOut != "CURRENTLY CLOCKED IN") {
                            totalHours += (Math.abs(adminUsers[i].times[j].clockOut.toDate() - adminUsers[i].times[j].clockIn.toDate()) / (1000 * 60 * 60));
                        }
                    }
                
                }
                hoursArray.push({
                    name: adminUsers[i].Name,
                    hours: Math.round(totalHours * 10) / 10,
                })
            }
            
        }

        console.log(hoursArray);
    }

    let getLocation = () => {
    return new Promise((resolve, reject) => {
      if (!navigator.geolocation) {
        console.error('Geolocation is not supported by your browser');
        resolve(null);
        return;
      }

      navigator.geolocation.getCurrentPosition(
        (position) => {
          const { latitude, longitude } = position.coords;
          resolve([latitude, longitude]);
        },
        (error) => {
          console.error('Error getting location', error);
          resolve(null);
        },
        {
          enableHighAccuracy: true,  // Try to use the most accurate location data available
          timeout: 5000,  // Wait 5 seconds for location data
          maximumAge: 0   // Do not use cached location data
        }
      );
    });
  };

    async function fetchLocation() {
        const location = await getLocation();
        if (location) {
            console.log('Latitude:', location[0], 'Longitude:', location[1]);
        } else {
            console.log('Location permission denied or error occurred');
        }
        
    }

    

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

    async function clockIn() {
        error = false;
        if (!currTodo) {
            error = true;
        }
        todoList = [...todoList, [new Date()]];
        clockInTimes = [...clockInTimes, new Date()];
        clockOutTimes = [...clockOutTimes, "CURRENTLY CLOCKED IN"];
        currTodo = "";

        let currentLocation = await getLocation();

        if (currentLocation == null) {
            alert("Unable to clock in, you must allow location access for this tool to work.");
            return;
        }

        

    
        if (timeMapArray.length != 0 && timeMapArray[timeMapArray.length - 1].clockOut == "CURRENTLY CLOCKED IN") {
            alert("Unable to clock in, you are already clocked in!");
            return;
        }

        let currentTimes = {
            clockIn: new Date(),
            clockOut: "CURRENTLY CLOCKED IN",
            totalTime: null,
            location: currentLocation,
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
            //console.log(documents);

            adminUsers = documents;

            /*
            for (let i = 0; i < documents.length; i++) {
                if (adminUsers[i].Name != null) {
                    adminUsersConverted.push(
                        {
                            email: documents[i].email,
                            id: documents[i].id,
                            times: documents[i].times,
                            Name: documents[i].Name
                        }
                    );

                    //console.log(adminUsersConverted[0].times);

                    for (let j = 0; j < adminUsersConverted[i].times.length; j++) {
                        //console.log(adminUsersConverted[i].times[j]);
                        if (adminUsersConverted[i].times[j] != null) {
                            adminUsersConverted[i].times[j].clockIn = adminUsersConverted[i].times[j].clockIn.toDate();
                        }
                        
                        if (adminUsersConverted[i].times[j].clockOut != null && adminUsersConverted[i].times[j].clockOut != "CURRENTLY CLOCKED IN") {
                            adminUsersConverted[i].times[j].clockOut = adminUsersConverted[i].times[j].clockOut.toDate();
                        }
                    }
                }
                
            }

            */

            console.log(adminUsersConverted)

            console.log(adminUsers);

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
        adminUsers = getAllUserTimes();
    }

    async function updateClockIn(userId, index, newDateTime) {
    try {
        const userRef = db.collection('users').doc(userId);
        const userDoc = await userRef.get();

        if (!userDoc.exists) {
            console.log('No such document!');
            return;
        }

        const userData = userDoc.data();
        if (!Array.isArray(userData.times) || index >= userData.times.length) {
            console.log('Invalid index');
            return;
        }

        userData.clockInArray[index].clockIn = newDateTime;

        await userRef.update({
            clockInArray: userData.clockInArray
        });

        console.log('Document successfully updated!');
    } catch (error) {
        console.error('Error updating document: ', error);
    }
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
            <h1>Hours Per Pay Period</h1>
            <Select class="mt-2" on:change={() => {calculatePayHours()}} items={periodArray} bind:value={selectedPeriod} />
            {#if hoursArray != []}
                    <table>
                        <tr>
                            <th>Name</th>
                            <th>Hours</th>
                        </tr>
                    {#each hoursArray as hoursItem}
                        <tr>
                            <td>{hoursItem.name}</td>
                            <td>{hoursItem.hours}</td>
                        </tr>
                    {/each}
                    </table>
                
            {/if}
            <hr>
                {#each adminUsers as user}
                    {#if user.Name != null}
                        <h1>{user.Name}</h1>
                        {#each user.times as clockInItem, index}
                        {#if (clockInItem.clockIn.toDate().getHours() != 5) &&  (clockInItem.clockIn.toDate().getHours() != 13)}
                                <div class="todo late">
                                    <div class="horiz-list">
                                        <DateInput  class={user.email + " " + index + " clockout"} value={clockInItem.clockIn.toDate()} /> - 
                                        {#if clockInItem.clockOut != "CURRENTLY CLOCKED IN"}
                                            <input value={clockInItem.clockIn.toDate().toISOString().substring(0,10)} type="date" />
                                            <input value={clockInItem.clockIn.toDate().toISOString().substring(11,16)} type="time" />
                                        {:else}
                                            <p>Currently Clocked In</p>
                                        {/if}
                                    </div>
                                    
                                    <div class="options">
                                        
                                        
                                        {#if clockInItem.location != null && clockInItem.location[0] != null && clockInItem.location[1] != null}
                                            <a href={"https://maps.google.com/?q=" + clockInItem.location[0] + "," + clockInItem.location[1]}>View Location</a>
                                        {/if}
                                    </div>
                                    
                                </div>
                            {:else}
                                <div class="todo">
                                    <p>
                                        {index + 1}. {clockInItem.clockIn.toDate().toLocaleDateString('en-US')} {clockInItem.clockIn.toDate().toLocaleTimeString('en-US')}   -   {clockInItem.clockOut == "CURRENTLY CLOCKED IN" ? "CURRENTLY CLOCKED IN" : clockInItem.clockOut.toDate().toLocaleDateString('en-us')} {clockInItem.clockOut == "CURRENTLY CLOCKED IN" ? "" : clockInItem.clockOut.toDate().toLocaleTimeString('en-us')}
                                    </p>
                                    {#if clockInItem.location != null && clockInItem.location[0] != null && clockInItem.location[1] != null}
                                        <a href={"https://maps.google.com/?q=" + clockInItem.location[0] + "," + clockInItem.location[1]}>View Location</a>
                                    {/if}
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

    .horiz-list {
        display: flex;
        gap: 25px;
    }
    .modal {
        position: fixed;
        top: 0;
        left: 0;
    }

    a, .button {
        background-color: transparent;
        color: white;
        font-size: 0.8rem;
        border: white solid;
        border-width: 2px;
        padding: 2px;
        border-radius: 2px;
    }

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
