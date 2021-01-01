<!-- App.svelte -->
<script>
    /*
    CRUD sessions.

    */
    import { ApolloClient, InMemoryCache, gql } from "@apollo/client";
    import { onMount } from "svelte";
    import { setClient } from "svelte-apollo";
    import { ApolloLink } from "apollo-link";
    import { createHttpLink } from "apollo-link-http";
    import { slide, fade } from "svelte/transition";

    import "./Tailwind.svelte";

    const httpLink = createHttpLink({
        uri: "https://polite-goldfish-60.hasura.app/v1/graphql",
    });

    const middlewareLink = new ApolloLink((operation, forward) => {
        operation.setContext({
            headers: {
                "x-hasura-admin-secret": "dro5c2Ur",
            },
        });
        return forward(operation);
    });

    const link = middlewareLink.concat(httpLink);

    function uuidv4() {
        return "xxxxxxxx-xxxx-4xxx-yxxx-xxxxxxxxxxxx".replace(
            /[xy]/g,
            function (c) {
                var r = (Math.random() * 16) | 0,
                    v = c == "x" ? r : (r & 0x3) | 0x8;
                return v.toString(16);
            }
        );
    }

    const defaultOptions = {
        watchQuery: {
            fetchPolicy: "no-cache",
            errorPolicy: "ignore",
        },
        query: {
            fetchPolicy: "no-cache",
            errorPolicy: "all",
        },
    };
    const client = new ApolloClient({
        uri: "https://polite-goldfish-60.hasura.app/v1/graphql",
        cache: new InMemoryCache({}),
        defaultOptions,

        link,
    });
    setClient(client);

    let exercise_groups = [];
    let exercises = [];
    // if the user has selected a session, this is an object with session information
    // g
    let sessions = [];
    let currentSession = null;
    let currentExercise = null;
    let exerciseSelector;
    function today() {
        const date = new Date();
        const year = date.getFullYear();
        const month = date.getMonth() + 1;
        const day = date.getDate().toString().padStart(2, "0");
        return `${year}-${month}-${day}`;
    }

    // new session
    let newSessionDate = today();
    let newSessionGroup;

    // new exercise
    let newExerciseName;
    let newExerciseNameInput;
    let newExerciseGroup;
    let newExerciseWeighted;
    let newExerciseReps;
    let newExerciseBreaths;
    let newExerciseTimed;
    let newExerciseDistance;

    // new performance
    let newWeight;
    let newReps;
    let newBreaths;
    let newTime;
    let newDistance;
    onMount(async () => {
        sessions = await getSessions();
        const lookups = await getLookups();
        exercise_groups = lookups.exercise_groups;
        exercises = lookups.exercises;
        currentSession = sessions[sessions.length - 1];
    });

    function handleSetCurrentExercise(event) {
        const newId = event.target.value;
        if (currentExercise && currentExercise.id === newId) {
            return;
        }

        currentExercise = exercises.find((e) => e.id === newId);
    }

    async function getLookups() {
        const result = await client.query({
            query: gql`
                query {
                    exercise_groups {
                        id
                        name
                    }
                    exercises(order_by: { name: asc }) {
                        id
                        name
                        weighted
                        reps
                        breaths
                        timed
                        distance
                        exercise_group {
                            id
                            name
                        }
                    }
                }
            `,
        });
        return result.data;
    }

    async function getSessions() {
        const result = await client.query({
            query: gql`
                query {
                    exercise_sessions {
                        id
                        date
                        created_at
                        updated_at
                        exercise_group {
                            id
                            name
                        }
                        exercise_performances {
                            id
                            exercise {
                                name
                            }
                            weight
                            reps
                            breaths
                            time
                            distance
                        }
                    }
                }
            `,
        });
        return result.data.exercise_sessions;
    }
    async function createSession({ exercise_group_id, date }) {
        /*
        Returns the id of the session
        */
        const id = uuidv4();
        // optimistically create one
        let session = {
            id,
            date,
            created_at: null,
            updated_at: null,
            exercise_group: {
                id: exercise_group_id,
                name: exercise_groups.find((g) => g.id === exercise_group_id)
                    .name,
            },
            exercise_performances: [],
        };
        sessions = sessions.concat(session);
        const result = await client.mutate({
            mutation: gql`
                mutation insert_session(
                    $object: exercise_sessions_insert_input!
                ) {
                    insert_exercise_sessions_one(object: $object) {
                        id
                        date
                        created_at
                        updated_at
                        exercise_group {
                            id
                            name
                        }
                        exercise_performances {
                            id
                            exercise {
                                name
                            }
                            weight
                            breaths
                            reps
                            time
                            distance
                        }
                    }
                }
            `,
            variables: {
                object: { exercise_group_id, date, id },
            },
        });
        // now correct with data from server
        session = result.data.insert_exercise_sessions_one;
        sessions = sessions.map((s) => {
            if (s.id === id) {
                return session;
            }
            return s;
        });
        return session;
    }

    async function deleteSession(id) {
        // optimistically delete
        sessions = sessions.filter((s) => s.id !== id);
        const result = await client.mutate({
            mutation: gql`
                mutation delete_exercise_session($id: uuid!) {
                    delete_exercise_sessions_by_pk(id: $id) {
                        id
                    }
                }
            `,
            variables: {
                id,
            },
        });
        return result.data.delete_exercise_sessions_by_pk;
    }

    async function addExercisePerformance(object) {
        const id = uuidv4();
        object = {
            ...object,
            id,
        };
        let performance = {
            id,
            exercise: {
                id: object.exercise_id,
                name: exercises.find((e) => e.id === object.exercise_id).name,
            },
            weight: object.weight,
            reps: object.reps,
            breaths: object.breaths,
            time: object.time,
            distance: object.distance,
        };
        currentSession.exercise_performances = currentSession.exercise_performances.concat(
            performance
        );
        const result = await client.mutate({
            mutation: gql`
                mutation add_performance(
                    $object: exercise_performances_insert_input!
                ) {
                    insert_exercise_performances_one(object: $object) {
                        id
                        exercise {
                            name
                        }
                        weight
                        reps
                        breaths
                        time
                        distance
                    }
                }
            `,
            variables: {
                object,
            },
        });
        performance = result.data.insert_exercise_performances_one;
        currentSession.exercise_performances.map((p) => {
            if (p.id === id) {
                return performance;
            }
            return p;
        });
        return performance;
    }

    async function deleteExercisePerformance(id) {
        currentSession.exercise_performances = currentSession.exercise_performances.filter(
            (p) => p.id !== id
        );
        const result = await client.mutate({
            mutation: gql`
                mutation delete_performance($id: uuid!) {
                    delete_exercise_performances_by_pk(id: $id) {
                        id
                    }
                }
            `,
            variables: { id },
        });
        return result.delete_exercise_performances_by_pk;
    }

    async function addExercise(object) {
        console.log(object, exercises);
        if (
            exercises.find(
                (e) =>
                    e.name === object.name &&
                    e.exercise_group.id === object.exercise_group_id
            )
        ) {
            alert("Already have that exercise");
            return;
        }
        const id = uuidv4();
        object = {
            ...object,
            id,
        };
        let exercise = {
            id,
            exercise_group: {
                id: object.exercise_group_id,
                name: exercise_groups.find(
                    (e) => e.id === object.exercise_group_id
                ).name,
            },
            name: object.name,
            weighted: object.weighted,
            reps: object.reps,
            breaths: object.breaths,
            timed: object.timed,
            distance: object.distance,
        };
        exercises = exercises.concat(exercise);
        const result = await client.mutate({
            mutation: gql`
                mutation add_exercise($object: exercises_insert_input!) {
                    insert_exercises_one(object: $object) {
                        id
                        name
                        exercise_group {
                            id
                            name
                        }
                        weighted
                        reps
                        breaths
                        timed
                        distance
                    }
                }
            `,
            variables: {
                object,
            },
        });
        exercise = result.data.insert_exercises_one;
        exercises = exercises.map((e) => {
            if (e.id === id) {
                return exercise;
            }
            return e;
        });
        return exercise;
    }

    async function deleteExercise(id) {
        exercises = exercises.filter((p) => p.id !== id);
        const result = await client.mutate({
            mutation: gql`
                mutation delete_exercise($id: uuid!) {
                    delete_exercises_by_pk(id: $id) {
                        id
                    }
                }
            `,
            variables: { id },
        });
        return result.delete_exercises_by_pk;
    }
</script>

<style global>
    body {
        @apply dark:bg-gray-900 dark:text-gray-100;
    }
    .input {
        @apply border dark:border-gray-700 dark:bg-gray-800 rounded p-2;
    }
    .input:disabled {
        @apply dark:border-transparent dark:bg-gray-900;
    }
    .checkbox {
        @apply border rounded w-8 h-8;
    }
    .delete-button {
        @apply bg-red-600 py-2 w-24 border dark:border-transparent dark:hover:bg-red-500 rounded outline-none;
    }
    .add-button {
        @apply bg-green-500 py-2 w-24 border dark:border-gray-700 dark:hover:bg-green-400 rounded outline-none;
    }
    .line {
        @apply border-t dark:border-gray-700 hover:bg-gray-700 cursor-pointer;
    }
    .line-selected {
        @apply bg-gray-800;
    }
</style>

<main>
    <!--<h1 class="text-light text-white text-xl text-center m-8">MyFit</h1>-->
    {#if currentSession}
        <h2 class="text-2xl text-center text-thin m-4">
            Current Session
            <br />{currentSession.exercise_group.name}
            {currentSession.date}
        </h2>
        <table class="w-auto m-auto">
            <tr>
                <th class="p-2">Exercise</th>
                <th class="p-2">Weight</th>
                <th class="p-2">Reps</th>
                <th class="p-2">Breaths</th>
                <th class="p-2">Time</th>
                <th class="p-2">Distance</th>
                <th />
            </tr>
            {#each currentSession.exercise_performances as performance (performance.id)}
                <tr transition:fade class="line">
                    <td class="p-2">{performance.exercise.name}</td>
                    <td class="p-2">{performance.weight}lbs</td>
                    <td class="p-2">{performance.reps}</td>
                    <td class="p-2">{performance.breaths}</td>
                    <td class="p-2">{performance.time}</td>
                    <td class="p-2">{performance.distance}</td>
                    <td>
                        <button
                            class="delete-button"
                            on:click={() => deleteExercisePerformance(performance.id)}>Delete</button>
                    </td>
                </tr>
            {/each}
            <tr>
                <td class="p-2">
                    <select
                        on:change={handleSetCurrentExercise}
                        on:blur={handleSetCurrentExercise}
                        class="input">
                        <option disabled selected>Select exercise</option>
                        {#each exercises as exercise (exercise.id)}
                            <option value={exercise.id}>{exercise.name}</option>
                        {/each}
                    </select>
                </td>
                <td class="p-2">
                    <input
                        class="input"
                        type="text"
                        bind:value={newWeight}
                        disabled={!currentExercise || !currentExercise.weighted} />
                </td>
                <td class="p-2">
                    <input
                        class="input"
                        type="text"
                        bind:value={newReps}
                        disabled={!currentExercise || !currentExercise.reps} />
                </td>
                <td class="p-2">
                    <input
                        class="input"
                        type="text"
                        bind:value={newBreaths}
                        disabled={!currentExercise || !currentExercise.breaths} />
                </td>
                <td class="p-2">
                    <input
                        class="input"
                        type="text"
                        bind:value={newTime}
                        disabled={!currentExercise || !currentExercise.timed} />
                </td>
                <td class="p-2">
                    <input
                        class="input"
                        type="text"
                        bind:value={newDistance}
                        disabled={!currentExercise || !currentExercise.distance} />
                </td>
                <td>
                    <button
                        class="add-button"
                        on:click={async () => {
                            await addExercisePerformance({
                                exercise_session_id: currentSession.id,
                                exercise_id: currentExercise.id,
                                weight: newWeight || 0,
                                reps: newReps || 0,
                                breaths: newBreaths || 0,
                                time: newTime || '0:00',
                                distance: newDistance || 0,
                            });
                        }}>Add</button>
                </td>
            </tr>
        </table>
    {/if}
    <hr class="my-8 border-gray-800" />
    <h2 class="text-2xl text-center text-thin m-4">Manage Sessions</h2>
    <table class="w-auto m-auto">
        <tr>
            <th class="p-2">Date</th>
            <th class="p-2">Exercise Group</th>
            <th />
        </tr>
        {#each sessions as session (session.id)}
            <tr
                transition:fade
                class="line"
                class:line-selected={currentSession && currentSession.id === session.id}
                on:click={() => (currentSession = session)}>
                <td class="p-2">{session.date}</td>
                <td class="p-2">{session.exercise_group.name}</td>
                <td class="p-2">
                    <button
                        class="delete-button"
                        on:click={async () => {
                            deleteSession(session.id);
                            if (currentSession.id === session.id) {
                                if (sessions.length > 0) {
                                    currentSession = sessions[sessions.length - 1];
                                } else {
                                    currentSession = null;
                                }
                            }
                        }}>Delete</button>
                </td>
            </tr>
        {/each}
        <tr class="line">
            <td class="p-2">
                <input type="date" class="input" bind:value={newSessionDate} />
            </td>
            <td class="p-2">
                <select bind:value={newSessionGroup} class="input">
                    <option value="unselected" disabled selected>
                        Select group
                    </option>
                    {#each exercise_groups as group (group.id)}
                        <option value={group.id}>{group.name}</option>
                    {/each}
                </select>
            </td>
            <td class="p-2">
                <button
                    class="add-button"
                    on:click={async () => {
                        currentSession = await createSession({
                            exercise_group_id: newSessionGroup,
                            date: newSessionDate,
                        });
                    }}>Add</button>
            </td>
        </tr>
    </table>
    <!-- Exercises -->
    <hr class="my-8 border-gray-800" />
    <h2 class="text-2xl text-center text-thin m-4">Manage Exercises</h2>
    <table class="w-auto m-auto">
        <tr>
            <th class="text-left p-2">Exercise</th>
            <th class="text-left p-2">Group</th>
            <th class="p-2 w-24">Weighted</th>
            <th class="p-2 w-24">Reps</th>
            <th class="p-2 w-24 ">Breaths</th>
            <th class="p-2 w-24">Timed</th>
            <th class="p-2 w-24">Distance</th>
            <th />
        </tr>
        {#each exercises as exercise (exercise.id)}
            <tr transition:fade class="border-t dark:border-gray-700">
                <td class="p-2">{exercise.name}</td>
                <td class="p-2">{exercise.exercise_group.name}</td>
                <td class="text-center p-2">
                    <input
                        type="checkbox"
                        disabled
                        class="checkbox"
                        checked={exercise.weighted} />
                </td>
                <td class="text-center p-2">
                    <input
                        type="checkbox"
                        disabled
                        class="checkbox"
                        checked={exercise.reps} />
                </td>
                <td class="text-center p-2">
                    <input
                        type="checkbox"
                        disabled
                        class="checkbox"
                        checked={exercise.breaths} />
                </td>
                <td class="text-center p-2">
                    <input
                        type="checkbox"
                        disabled
                        class="checkbox"
                        checked={exercise.timed} />
                </td>
                <td class="text-center p-2">
                    <input
                        type="checkbox"
                        disabled
                        class="checkbox"
                        checked={exercise.distance} />
                </td>
                <td>
                    <button
                        class="delete-button"
                        on:click={() => window.confirm('Delete all logs for this exercise?') && deleteExercise(exercise.id)}>Delete</button>
                </td>
            </tr>
        {/each}
        <tr transition:slide class="border-t dark:border-gray-700">
            <td class="p-2">
                <input
                    bind:this={newExerciseNameInput}
                    type="text"
                    bind:value={newExerciseName}
                    class="input"
                    placeholder="Name" />
            </td>
            <td class="p-2">
                <select bind:value={newExerciseGroup} class="input">
                    <option value="unselected" disabled selected>
                        Select group
                    </option>
                    {#each exercise_groups as group (group.id)}
                        <option value={group.id}>{group.name}</option>
                    {/each}
                </select>
            </td>
            <td class="text-center p-2">
                <input
                    type="checkbox"
                    class="checkbox"
                    bind:checked={newExerciseWeighted} />
            </td>
            <td class="text-center p-2">
                <input
                    type="checkbox"
                    class="checkbox"
                    bind:checked={newExerciseReps} />
            </td>

            <td class="text-center p-2">
                <input
                    type="checkbox"
                    class="checkbox"
                    bind:checked={newExerciseBreaths} />
            </td>
            <td class="text-center p-2">
                <input
                    type="checkbox"
                    class="checkbox"
                    bind:checked={newExerciseTimed} />
            </td>
            <td class="text-center p-2">
                <input
                    type="checkbox"
                    class="checkbox"
                    bind:checked={newExerciseDistance} />
            </td>
            <td>
                <button
                    class="add-button"
                    on:click={() => {
                        addExercise({
                            name: newExerciseName,
                            exercise_group_id: newExerciseGroup,
                            weighted: !!newExerciseWeighted,
                            reps: !!newExerciseReps,
                            breaths: !!newExerciseBreaths,
                            timed: !!newExerciseTimed,
                            distance: !!newExerciseDistance,
                        });
                        newExerciseName = '';
                        newExerciseNameInput.focus();
                    }}>Add</button>
            </td>
        </tr>
    </table>
</main>
