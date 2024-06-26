<script lang="ts">
  import { FontAwesomeIcon } from '@fortawesome/svelte-fontawesome'
  import { faArrowAltCircleRight } from '@fortawesome/free-solid-svg-icons'
  import ThemeSwitcher from '$lib/ThemeSwitcher.svelte';
  import SvelteMarkdown from 'svelte-markdown'
  import autosize from 'svelte-autosize';
  import NewChat from "$lib/NewChat.svelte";
  import { browser } from "$app/environment";
  import { onMount } from "svelte";
  import * as aesjs from 'aes-js';
  import { writable } from "svelte/store";

  let messageText: string;

  let showNewModal: boolean = false;

  function handleKeyDown(event: KeyboardEvent): void {
    if (event.key === "Enter" && !event.shiftKey) {
      event.preventDefault();
      handleSubmit();
    }
  }

  interface User {
    nickname: string;
    key: string;
  }

  interface Message {
    timestamp: number;
    sender: string;
    receiver: string;
    encrypted: boolean;
    content: string;
  }

  let messagesStore = writable<Message[]>([]);
  let users: User[] = [];
  let messages: Message[] = [];
  let selectedUser: User | null = null;
  $: relevantMessages = selectedUser ? $messagesStore.filter(message => message.sender === selectedUser!.nickname || message.receiver === selectedUser!.nickname) : [];
  $: sortedMessages = relevantMessages.sort((a, b) => a.timestamp - b.timestamp);
  $: decryptedMessages = sortedMessages.map(message => decryptMessage(message, selectedUser));

  function decryptMessage(message: Message, user: User | null): Message {
    if (!user) {
      return message;
    }
    if (message.encrypted) {
      let key = Uint8Array.from(atob(user.key), c => c.charCodeAt(0));
      var aesCtr = new aesjs.ModeOfOperation.ctr(key, new aesjs.Counter(1));
      var encryptedBytes = aesjs.utils.hex.toBytes(message.content);
      var decryptedBytes = aesCtr.decrypt(encryptedBytes);
      var decryptedText = aesjs.utils.utf8.fromBytes(decryptedBytes);
      return {
        encrypted: false,
        content: decryptedText,
        timestamp: message.timestamp,
        sender: message.sender,
        receiver: message.receiver
      }
    }
    return message;
  }

  let db;
  let socket: WebSocket;

  function handleSubmit(): void {
    if (!messageText || !(messageText = messageText.trim()) || !selectedUser) {
      return
    }
    let key = Uint8Array.from(atob(selectedUser.key), c => c.charCodeAt(0));
    var aesCtr = new aesjs.ModeOfOperation.ctr(key, new aesjs.Counter(1));
    var textBytes = aesjs.utils.utf8.toBytes(messageText);
    var encryptedBytes = aesCtr.encrypt(textBytes);
    var encryptedHex = aesjs.utils.hex.fromBytes(encryptedBytes);
    let message = {
      encrypted: true,
      content: encryptedHex,
      timestamp: Date.now(),
      receiver: selectedUser.nickname,
      sender: data.nickname
    }
    console.log(selectedUser);
    let payload = {
      type: 'message',
      data: message
    }
    socket.send(JSON.stringify(payload));
    messageText = '';
    return;
  }
  
  export let data: { nickname: string };

  let markedDirty = false;
  
  if (browser) {
    onMount(() => {
      socket = new WebSocket('ws://localhost:3000');
      socket.onopen = () => {
        console.log('Connected to server');
        let body = {
          type: 'nickname',
          data: {
            nickname: data.nickname
          }
        };
        socket.send(JSON.stringify(body));
      };

      const request = indexedDB.open("messages", 2);
      
      request.onupgradeneeded = function() {
        const db = request.result;
        var messageStore = db.createObjectStore("messages", { keyPath: "timestamp" });
        messageStore.createIndex("sender", "sender", { unique: false });
        messageStore.createIndex("receiver", "receiver", { unique: false });
        messageStore.createIndex("encrypted", "encrypted", { unique: false });
        messageStore.createIndex("content", "content", { unique: false });
        var userStore = db.createObjectStore("users", { keyPath: "nickname" });
        userStore.createIndex("key", "key", { unique: false });
      };

      request.onsuccess = function() {
        db = request.result;
        
        var transaction = db.transaction(["users"], "readonly");
        var userStore = transaction.objectStore("users");
        var getAllUsersRequest = userStore.getAll();
        getAllUsersRequest.onsuccess = function(event) {
          var usersRes = event.target?.result;
          users = usersRes ?? [];
          if (users.length > 0) {
            selectedUser = users[0];
          }
          console.log("Users retrieved successfully:", users);
        };

        var transaction2 = db.transaction(["messages"], "readonly");
        var messageStore = transaction2.objectStore("messages");
        var getAllMessagesRequest = messageStore.getAll();
        getAllMessagesRequest.onsuccess = function(event) {
          var messagesRes = event.target?.result;
          messagesStore.set(messagesRes ?? []);
          console.log("Messages retrieved successfully:", messagesStore);
        };
      };

      socket.onmessage = (event) => {
        let message = JSON.parse(event.data);
        let { type, data } = message;
        console.log('Received message:', message);
        if (type === 'conversation') {
          var transaction = db.transaction(["users"], "readwrite");
          var userStore = transaction.objectStore("users");
          let user = {
            "nickname": data.recipient,
            "key": data.key
          }
          console.log(user);
          var addRequest = userStore.add(user);
          addRequest.onsuccess = function(event) {
            console.log("New user added successfully");
            users.push(user);
            window.location.reload();
          };
          addRequest.onerror = function(event) {
            console.error("Error adding user:", event.target.error);
          };

            console.log('Users:', users)
          } else if (type === 'message') {
            var transaction = db.transaction(["messages"], "readwrite");
            var messageStore = transaction.objectStore("messages");
            let message = data.message;
            console.log(`Message: ${JSON.stringify(message)}`);
            var addRequest = messageStore.add(message);
            addRequest.onsuccess = function(event) {
              console.log("New message added successfully");
              messagesStore.update((mess) => {
                mess.push(message);
                markedDirty = !markedDirty;
                return mess;
              });
            };
            addRequest.onerror = function(event) {
              console.error("Error adding message:", event.target.error);
            };
          }
      };
      });
    }

    function newConversation(arg0: string) {
      console.log('New conversation:', arg0);
      socket.send(JSON.stringify({
        type: 'conversation',
        data: {
          recipientNick: arg0
        }
    }));
  }

</script>

<svelte:head>
  <title>Chat</title>
</svelte:head>

<div class="flex flex-col mx-auto p-2 h-screen">
  <div class="flex flex-row justify-between bg-gradient-to-r from-pink-500 to-purple-600 p-4 rounded-lg">
    <h1 class="text-4xl font-bold text-white">Quantum Peers</h1>
    <ThemeSwitcher />
  </div>

  <div class="flex flex-row flex-1 gap-2 overflow-auto h-screen">

    <div class="flex flex-col border-r-4 dark:border-neutral-950 pr-2">
      <div class="flex flex-col overflow-auto flex-1 mt-5">
        {#key markedDirty}
          {#each users as user}
            <button
              class="flex flex-row gap-5 sm:min-w-72 items-center p-2 rounded-md"
              class:bg-sky-500={selectedUser?.nickname === user.nickname}
              class:dark:bg-sky-600={selectedUser?.nickname === user.nickname}
              class:odd:dark:bg-neutral-900={selectedUser?.nickname !== user.nickname}
              class:even:dark:bg-neutral-800={selectedUser?.nickname !== user.nickname}
              class:odd:bg-neutral-100={selectedUser?.nickname !== user.nickname}
              class:even:bg-neutral-200={selectedUser?.nickname !== user.nickname}
              class:hover:cursor-default={selectedUser?.nickname === user.nickname}
              on:click={() => selectedUser = user}
              >
              <div class="text-5xl">👤</div>
              <div class="text-xl">{user.nickname}</div>
            </button>
          {/each}
        {/key}
      </div>

      <button class="p-4 mt-5 mb-3 bg-sky-500 dark:bg-sky-600 hover:bg-sky-600 dark:hover:bg-sky-700 transition rounded-2xl" on:click={() => showNewModal = true}
        >New Chat</button>
    </div>
    
    <div class="flex flex-col flex-1 overflow-scroll">
      <div class="flex flex-col flex-1 overflow-scroll gap-1 text-lg mt-5">
          {#each decryptedMessages as message}
            <div class="flex justify-between" class:flex-row-reverse={message.sender === data.nickname}>
              <div
                class="max-w-md px-3 py-1 rounded-lg shadow transition"
                class:bg-neutral-300={message.sender !== data.nickname}
                class:dark:bg-neutral-600={message.sender !== data.nickname}
                class:bg-sky-400={message.sender === data.nickname}
                class:dark:bg-sky-500={message.sender === data.nickname}
              >
                <SvelteMarkdown source={message.content} />
              </div>
            </div>
          {/each}
      </div>

      <form class="flex flex-row items-end " on:submit|preventDefault={handleSubmit}>
        <textarea class="max-h-32 flex-1 dark:bg-neutral-900 dark:border-neutral-700 border border-neutral-300 rounded-md w-full my-4 text-xl p-2"
          on:keydown={handleKeyDown}
          rows="1"
          placeholder="Send a message..."
          use:autosize
          bind:value={messageText}/>
        <button
          type="submit"
          class="rounded-3xl ml-2 my-4 transition text-5xl text-sky-500 hover:text-sky-600"
        >
          <FontAwesomeIcon icon={faArrowAltCircleRight} /> 
        </button>
      </form>
    </div>
  </div>
</div>

<NewChat bind:show={showNewModal} callback={newConversation} />
