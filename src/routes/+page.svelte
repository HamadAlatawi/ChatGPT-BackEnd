<script lang="ts">
    import ChatMessage from '$lib/components/ChatMessage.svelte'
    import type { ChatCompletionRequestMessage } from 'openai'
    import { SSE } from 'sse.js'

    let query: string = ''
    let answer: string = ''
    let loading: boolean = false
    //Store the messages here on the client side, no persistence.
    let chatMessages: ChatCompletionRequestMessage[] = []
    let scrollToDiv: HTMLDivElement

    function scrollToBottom() {
        setTimeout(function () {
            scrollToDiv.scrollIntoView({ behavior: 'smooth', block: 'end', inline: 'nearest' })
        }, 100)
    }

    const handleSubmit = async () => {
        loading = true
        //All messages in the chat put in the array and then concatenate the last user prompt
        chatMessages = [...chatMessages, { role: 'user', content: query }]

        //Create a POST request to the API endpoint
        const eventSource = new SSE('https://Domain-URL/api/gpt', {
            headers: {
                'Content-Type': 'application/json'
            },
            payload: JSON.stringify({ messages: chatMessages })
        })

        query = ''

        eventSource.addEventListener('error', handleError)

        //As the tokens are generated they are rendered onto the users screen. Parses the response as the message is coming
        //Because stream is set to true on the GPT agent in the api/chat/+server.ts
        eventSource.addEventListener('message', (e) => {
            scrollToBottom()
            try {
                loading = false
                if (e.data === '[DONE]') {
                    chatMessages = [...chatMessages, { role: 'assistant', content: answer }]
                    answer = ''
                    return
                }

                const completionResponse = JSON.parse(e.data)
                const [{ delta }] = completionResponse.choices

                if (delta.content) {
                    answer = (answer ?? '') + delta.content
                }
            } catch (err) {
                handleError(err)
            }
        })
        eventSource.stream()
        scrollToBottom()
    }

    //If a error occurs. A generic ts function is set. All values are reset to default
    function handleError<T>(err: T) {
        loading = false
        query = ''
        answer = ''
        console.error(err)
    }
</script>

<div class="navbar bg-base-100">
    <a class="btn btn-ghost normal-case text-xl">Mosaed.</a>
</div>
<div class="flex flex-col pt-4 w-full px-8 items-center gap-2">
    <div>
        <h1 class="text-2xl font-bold w-full text-center">ChatBot Hackathon Test</h1>
        <p class="text-sm italic">Current model GPT 3.5 turbo</p>
    </div>
    <div class="h-[500px] w-full bg-gray-900 rounded-md p-4 overflow-y-auto flex flex-col gap-4">
        <div class="flex flex-col gap-2">
            <ChatMessage type="assistant" message="Hello, ask me anything you want!" />
            {#each chatMessages as message}
                <ChatMessage type={message.role} message={message.content} />
            {/each}
            {#if answer}
                <ChatMessage type="assistant" message={answer} />
            {/if}
            {#if loading}
                <ChatMessage type="assistant" message="Loading..." />
            {/if}
        </div>
        <div class="" bind:this={scrollToDiv} />
    </div>
    <form
            class="flex w-full rounded-md gap-4 bg-gray-900 p-4"
            on:submit|preventDefault={() => handleSubmit()}
    >
        <input type="text" class="input input-bordered w-full" bind:value={query} />
        <button type="submit" class="btn btn-accent"> Send </button>
    </form>
</div>