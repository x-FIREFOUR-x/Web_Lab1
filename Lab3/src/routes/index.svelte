<script context="module">
	export const ssr = false;
</script>

<script>
	import formSpinner from '$lib/formSpinner.gif';
	import Task from './task.svelte';
	import { storeFE, idtask, showSpinner, showeror } from './store.js';
	import { fetchGraphQL, errorHandler, operationsDoc, insert } from './GraphQL.js';
	import { Client, createClient, defaultExchanges, subscriptionExchange } from '@urql/core';
	import { setClient } from '@urql/svelte';
	import { createClient as createWSClient } from 'graphql-ws';

	const wsClient = createWSClient({
		url: import.meta.env.VITE_API_WSS_ENDPOINT,
		connectionParams: {
			headers: {
				'x-hasura-admin-secret': import.meta.env.VITE_API_HASURA_ADMIN_SECRET
			}
		}
	});

	const client = createClient({
		url: import.meta.env.VITE_API_HTTPS_ENDPOINT,
		headers: { 'x-hasura-admin-secret': import.meta.env.VITE_API_HASURA_ADMIN_SECRET },
		exchanges: [
			...defaultExchanges,
			subscriptionExchange({
				forwardSubscription: (operation) => ({
					subscribe: (sink) => ({
						unsubscribe: wsClient.subscribe(operation, sink)
					})
				})
			})
		]
	});

	setClient(client);
	const sub = operationStore(`
		subscription MySub {
  			Tasks(order_by: {date: asc}) {
    			id
    			taskText
  			}
		}
	`);

	import { operationStore, subscription } from '@urql/svelte';

	const handleSubscription = (sub = [], data) => {
		$storeFE = [...data.Tasks];
		return [data.Tasks, ...sub];
	};

	$storeFE = [];
	let text = '';
	downloadTasks();

	function addTask() {
		if (text.trim()) {
			$showSpinner = true;
			fetchGraphQL(insert, 'MyMutation', { taskText: text })
				.then(function (v) {
					idtask.set(v.data.insert_Tasks_one.id);
					let size = $storeFE.length;
					$storeFE[size] = { id: $idtask, name: 'todo', taskText: text };
					text = '';
				})
				.catch(errorHandler)
				.finally(() => {
					$showSpinner = false;
				});
		}
	}

	function downloadTasks() {
		$showSpinner = true;
		fetchGraphQL(operationsDoc)
			.then((data) => {
				storeFE.set(data.data.Tasks);
			})
			.catch(errorHandler)
			.finally(() => {
				$showSpinner = false;
			});
		subscription(sub, handleSubscription);
	}

	let offline = false;
	window.onoffline = () => {
		offline = true;
	};
	window.ononline = () => {
		offline = false;
	};

	let form;
</script>

{#if offline}
	<h1>Ошибка перевірте інтернет підключення</h1>
{:else if !offline}
	{#if $showeror}
		<h1>Ошибка</h1>
	{:else}
		<form id="form" method="post" bind:this={form} on:submit|preventDefault={addTask}>
			<h1>
				<i><u>Завдання</u></i>
			</h1>
			<input
				id="add"
				type="text"
				name="nametask"
				maxlength="25"
				bind:value={text}
				placeholder="Введіть ваше завдання"
			/>
			<button type="submit"> Добавити завдання </button>
			<ul id="tasks">
				{#each $storeFE as task (task.id)}
					<svelte:component this={Task} objAttributes={task} />
				{/each}
			</ul>
		</form>
		{#if $showSpinner}
			<img src={formSpinner} alt="spinner" />
		{/if}
	{/if}
{/if}

<style>
	* {
		margin: 2px;
		--bg-color: white;
		--border-color: black;
		--button-color: rgb(251, 255, 0);
		--button-border-color: rgb(255, 197, 5);
	}
	form {
		display: flex;
		flex-direction: column;
		justify-content: center;
		align-items: center;
		text-align: center;
		min-width: 100%;
		min-height: 100%;
	}
	h1 {
		background-color: var(--border-list);
		flex-direction: column;
		align-items: center;
	}
	input {
		background-color: var(--bg-color);
		border: 1px solid var(--border-color);
		font-size: 20px;
		padding: 3px;
		min-width: 70%;
		margin: 4px;
	}
	button {
		background-color: var(--button-color);
		border-color: var(--button-border-color);
		min-width: 50%;
		font-size: 15px;
		padding: 3px;
		margin: 4px;
	}
	ul {
		align-items: center;
		text-align: left;
		padding: 0;
		min-width: 100%;
	}
	img {
		top: 0;
		left: 0;
		position: fixed;
		z-index: 2;
		height: 100%;
		width: 100%;
		margin: 0;
		padding: 0;
	}
</style>
