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
    function today() {
        const date = new Date();
        return `${date.getFullYear()}-${date.getMonth()}-${date.getDay()}`;
    }
    let newSessionDate;
    let newSessionGroup;
    onMount(async () => {
        sessions = await getSessions();
        console.log(sessions);
        const lookups = await getLookups();
        exercise_groups = lookups.exercise_groups;
        exercises = lookups.exercises;
    });

    async function getLookups() {
        const result = await client.query({
            query: gql`
                query {
                    exercise_groups {
                        id
                        name
                    }
                    exercises {
                        id
                        name
                        weighted
                        breaths
                        reps
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
                    }
                }
            `,
            variables: {
                object: { exercise_group_id, date },
            },
        });
        return result.data.insert_exercise_sessions_one;
    }

    async function deleteSession(id) {
        const result = await client.mutate({
            mutation: gql`
                mutation delete_exercise_session($id: Int!) {
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
</script>

<style>
    /* css will go here */
</style>

<div class="App">
    <table>
        <tr>
            <th>ID</th>
            <th>Date</th>
            <th>Exercise Group</th>
        </tr>
        {#each sessions as session (session.id)}
            <tr>
                <td class="link">
                    <span
                        on:click={() => (currentSession = session)}>{session.id}</span>
                </td>
                <td>{session.date}</td>
                <td>{session.exercise_group.name}</td>
                <td class="link">
                    <span
                        on:click={async () => {
                            await deleteSession(session.id);
                            if (currentSession === session.id) {
                                currentSession = null;
                            }

                            sessions = await getSessions();
                            console.log('now have sessions', sessions);
                        }}>delete</span>
                </td>
            </tr>
        {/each}
        <tr>
            <td>(auto)</td>
            <td><input type="date" bind:value={newSessionDate} /></td>
            <td>
                <select bind:value={newSessionGroup}>
                    {#each exercise_groups as group (group.id)}
                        <option value={group.id}>{group.name}</option>
                    {/each}
                </select>
            </td>
            <td>
                <button
                    on:click={async () => {
                        const session = await createSession({
                            exercise_group_id: newSessionGroup,
                            date: newSessionDate,
                        });
                        sessions = await getSessions();
                        currentSession = session;
                    }}>Create</button>
            </td>
        </tr>
    </table>
    <select>
        {#each exercise_groups as group (group.id)}
            <option value={group.id}>{group.name}</option>
        {/each}
    </select>
    <header class="App-header">
        <a
            class="App-link"
            href="https://svelte.dev"
            target="_blank"
            rel="noopener noreferrer">
            Learn Svelte
        </a>
    </header>
</div>
