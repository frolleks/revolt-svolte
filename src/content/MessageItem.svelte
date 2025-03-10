<script lang="ts">
  import { client } from "Client";
  import { messageContext, showOptionContext } from "contextmenu/ContextMenus";
  import { floatingMenu, showMemberContext } from "contextmenu/FloatingMenu";
  import { DateTime } from "luxon";
  import { ModalStack } from "modals/ModalStack";
  import { Attachment, EmbedMedia, EmbedWeb, type BaseMessage, type Embed } from "revolt-toolset";
  import {
    HoveredMessage,
    isEditing,
    MessageCache,
    MobileLayout,
    selectBottom,
    SelectedChannel,
  } from "State";
  import { Theme } from "Theme";
  import { MessageDetails } from "utils";
  import MessageItemAttachment from "./MessageItemAttachment.svelte";
  import MessageItemContent from "./MessageItemContent.svelte";
  import MessageItemEmbed from "./MessageItemEmbed.svelte";
  import MessageItemHeader from "./MessageItemHeader.svelte";
  import MessageItemReplies from "./MessageItemReplies.svelte";
  import MessageItemToolbar from "./MessageItemToolbar.svelte";

  export let message: BaseMessage;

  let isReply = false,
    shouldSeparate = true,
    isHovered = false,
    doHighlight = false,
    embeds: (Embed | EmbedWeb)[] = [],
    attachments: (Attachment | EmbedMedia)[] = [];
  $: {
    const previousMessage =
      $MessageCache[$SelectedChannel!.id]?.[
        $MessageCache[$SelectedChannel!.id]?.indexOf(message) - 1
      ];
    isReply = message.isUser() && !!message.replies.length;
    shouldSeparate =
      !message.isUser() ||
      isReply ||
      !previousMessage ||
      !previousMessage.isUser() ||
      previousMessage.authorID !== message.authorID ||
      JSON.stringify(previousMessage.masquerade) !== JSON.stringify(message.masquerade) ||
      Math.abs(previousMessage.createdAt - message.createdAt) >= 420000;
    isHovered = $HoveredMessage == message.id && $isEditing !== message.id;
    doHighlight =
      message.isUser() && message.mentionIDs.includes(client.user.id) && $isEditing !== message.id;
    embeds = message.isUser()
      ? <any>message.embeds.filter((e) => e.isText() || (e.isWeb() && e.special?.type !== "GIF"))
      : [];
    attachments = message.isUser()
      ? [...(message.attachments || []), ...(<EmbedMedia[]>message.embeds
            .filter((e) => e.isMedia() || (e.isWeb() && e.special?.type == "GIF"))
            .map((e) =>
              e.isMedia()
                ? e
                : e.isWeb() &&
                  new EmbedMedia(e.client!, {
                    type: "Image",
                    width: e.source.image?.width || 0,
                    height: e.source.image?.height || 0,
                    url: e.source.image?.url || "",
                    size: e.source.image?.size || "Large",
                  })
            )
            .filter((e) => e))]
      : [];
  }

  let startScroll: number | null = null;
  function handleClickDown(e: TouchEvent) {
    const list = document.getElementById("MessageList");
    startScroll = list?.scrollTop ?? null;
  }
  function handleClick(e: TouchEvent | MouseEvent) {
    const target = e.target as HTMLElement;
    if (!$MobileLayout || !e.isTrusted) return (startScroll = null);
    if (
      target.tagName == "A" ||
      [...document.querySelectorAll("[data-clickable]")].find((c) => c.contains(target))
    ) {
      if (document.activeElement?.id == "Textbox") {
        e.preventDefault();
        target.click();
        selectBottom(true);
      }
      startScroll = null;
      return;
    }
    const list = document.getElementById("MessageList")!;
    if (startScroll === null || Math.abs(startScroll - list.scrollTop) <= 1)
      HoveredMessage.set(message.id);
    startScroll = null;
  }
</script>

{#if $SelectedChannel}
  <div class="flex flex-col gap-0.5">
    {#if isReply && message.isUser()}
      <MessageItemReplies {message} />
    {/if}
    <div
      class="relative px-1 [line-height:normal] {shouldSeparate ? 'mt-3' : ''}"
      style:background-color={isHovered
        ? $Theme["secondary-background"]
        : doHighlight
        ? $Theme["mention"]
        : ""}
      on:mouseenter={() => !$MobileLayout && HoveredMessage.set(message.id)}
      on:mousemove={() => !$MobileLayout && HoveredMessage.set(message.id)}
      on:mouseleave={() => !$MobileLayout && HoveredMessage.set(null)}
      on:wheel={() => !$MobileLayout && HoveredMessage.set(message.id)}
      on:touchstart={handleClickDown}
      on:touchend={handleClick}
      on:contextmenu={(e) => showOptionContext(e, messageContext(message))}
    >
      {#if message.isUser()}
        <div class="flex gap-2 {shouldSeparate ? '' : 'items-center'}">
          {#if shouldSeparate}
            <img
              class="rounded-full h-10 w-10 shrink-0 object-cover cursor-pointer"
              src={MessageDetails(message).avatar}
              alt=""
              data-clickable
              on:click={(e) => {
                if (!message.isUser()) return;
                if (message.member)
                  showMemberContext(message.member, e.clientX, e.clientY, e.target);
                else ModalStack.push({ type: "user", id: message.authorID });
              }}
              on:dblclick={() => {
                if (!message.isUser()) return;
                if ($floatingMenu) floatingMenu.set(null);
                ModalStack.push({ type: "user", id: message.authorID });
              }}
            />
          {:else}
            <div
              class="h-full w-10 shrink-0 text-center overflow-hidden whitespace-nowrap"
              style:font-size="0.65rem"
              style:color={$Theme["tertiary-foreground"]}
            >
              {#if isHovered}
                {DateTime.fromMillis(message.createdAt).toFormat("t")}
              {:else if message.edited}
                (edited)
              {/if}
            </div>
          {/if}
          <div
            class="flex flex-col flex-1 max-w-[calc(100%-3rem)] py-0.5 {!$MobileLayout
              ? 'select-text'
              : ''}"
          >
            {#if shouldSeparate}
              <MessageItemHeader {message} />
            {/if}
            {#if message.content || $isEditing == message.id}
              <MessageItemContent {message} />
            {/if}
            {#each attachments as attachment (attachment instanceof Attachment ? attachment.id : `${attachment.width}x${attachment.height}x${attachment.size + attachment.type}`)}
              <MessageItemAttachment {attachment} />
            {/each}
            {#each embeds as embed}
              <MessageItemEmbed {embed} />
            {/each}
          </div>
        </div>
      {/if}
      {#if isHovered}
        <MessageItemToolbar {message} />
      {/if}
    </div>
  </div>
{/if}
