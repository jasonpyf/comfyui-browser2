<script lang="ts">
  import { onMount } from 'svelte';
  import { onLoadWorkflow, onScroll, fetchFiles, WHITE_EXTS } from './utils';
  import { t } from 'svelte-i18n';
  import type { FOLDER_TYPES } from './utils';
  import MediaShow from './MediaShow.svelte';
  import type Toast from './Toast.svelte';

  export let comfyUrl: string;
  export let folderType: FOLDER_TYPES;
  export let toast: Toast;
  export let folderPath: string;
  $: if (folderPath != undefined) {
    refresh();
  }

  $: try {
    searchRegex = new RegExp(searchQuery.toLowerCase());
  } catch {
    searchRegex = new RegExp('');
  }

  let comfyApp: any;
  let files: Array<any> = [];
  let loaded: boolean = true;
  let uploadUrl = "http://nuwa.datawing.zhangyou.com";
  let uploadModal: any;
  let uploading: boolean = false;
  let showCursor = 20;
  let searchQuery = '';
  let searchRegex = new RegExp('');
  let scrollTop = 0;

  /** 表单项 **/
  let currentFile: any;
  let name: any;
  let gameId: any;
  let userId: any;
  let tags: any;

  let users: any[] = [];
  let games:any[] = [];

  $: tt = function(key: string) {
    return $t('filesList.' + key);
  }

  export async function refresh() {
    loaded = false;
    files = await fetchFiles(folderType, comfyUrl, folderPath);
    loaded = true;
  }

  export async function loadGameOptions() {
    const res = await fetch(uploadUrl + '/nuwa/workshop/v3/api-open/game/options', {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
      },
      body: "{}",
    });
    const ret = await res.json();
    if (ret.status === 200) {
      games = ret.data.list;
    }
  }

  export async function loadUserOptions() {
    const res = await fetch(uploadUrl + '/nuwa/workshop/v3/api-open/user/options', {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
      },
      body: "{}",
    });
    const ret = await res.json();
    if (ret.status === 200) {
      users = ret.data.list;
    }
  }

  export async function showUploadModal(file:any) {
    currentFile = file;

    // 重置表单
    // name = null;
    // gameId = null;
    // userId = null;
    // tags = null;

    await loadUserOptions();
    await loadGameOptions();
    uploadModal.showModal();
  }

  export async function uploadFile() {
    if (!name) {
      toast.show(false, "", "素材名称不能为空！");
      return;
    }
    if (!gameId) {
      toast.show(false, "", "游戏项目不能为空！");
      return;
    }
    if (!userId) {
      toast.show(false, "", "制作人不能为空！");
      return;
    }

    uploading = true;

    try {
      const fileData = await fetch(currentFile.url);
      const fileBlob = await fileData.blob();
      const file = new File([fileBlob], currentFile.name, { type: fileBlob.type });
      const formData = new FormData();
      formData.append('name', name); 
      formData.append('gameId', gameId); 
      formData.append('nickname', userId); 
      formData.append('tags', tags); 
      formData.append('file', file); 
      const res = await fetch(uploadUrl + '/nuwa/workshop/v3/api-open/ai/material/upload', {
        method: 'POST',
        body: formData,
      });
      const ret = await res.json();
      if (ret.status === 200) {
        toast.show(true, ret.message, "");
        uploadModal.close();
      } else {
        toast.show(false, "", ret.error);
        uploadModal.close();
      }
    } catch (e) {
      toast.show(false, "", `{e}`);
      uploadModal.close();
    }

    uploading = false;
  }

  onMount(async () => {
    //@ts-ignore
    comfyApp = window.top.app;

    //@ts-ignore
    window.top.addEventListener("comfyuiBrowserShow", () => {
      refresh();
    });

    folderPath = '';

    window.addEventListener('scroll', (e) => {
      showCursor = onScroll(showCursor, files.length);
      //@ts-ignore
      scrollTop = (e.target.scrollingElement as HTMLElement).scrollTop;
    });
  });

  async function onCollect(file: any) {
    const res = await fetch(comfyUrl + '/browser/collections', {
      method: 'POST',
      body: JSON.stringify({
        filename: file.name,
        folder_path: file.folder_path,
        folder_type: folderType,
      }),
    });

    toast.show(
      res.ok,
      tt('fileList.Added to Saves'),
      tt('fileList.Failed to add to Saves'),
    );
  }

  async function scrollToTop() {
    window.scrollTo(0, 0);
  }


  async function onDelete(file: any) {
    const ret = confirm(tt('You want to delete this file?') + ' ' + file.name);
    if (!ret) {
      return;
    }

    const res = await fetch(comfyUrl + '/browser/files', {
      method: 'DELETE',
      body: JSON.stringify({
        folder_type: folderType,
        filename: file.name,
        folder_path: file.folder_path,
      }),
    });

    refresh();
    toast.show(
      res.ok,
      tt('Deleted the file') + file.name,
      tt('Failed to delete the file'),
    );
  }

  async function onClickDir(dir: any) {
    folderPath = dir.path;
  }

  async function onClickPath(index: number) {
    if (index === -1) {
      folderPath = '';
      return;
    }

    folderPath = folderPath
      .split('/')
      .slice(0, index + 1)
      .join('/');
  }
</script>

<div class="max-w-full text-sm breadcrumbs flex flex-row ml-4">
  <ul class="basis-3/4">
    <li>
      <button on:click={() => onClickPath(-1)}>{$t('common.rootDir')}</button>
    </li>
    {#each (folderPath || '').split('/') as path, index}
      <li><button on:click={() => onClickPath(index)}>{path}</button></li>
    {/each}
  </ul>

  <input
    type="text"
    placeholder={tt('searchInput.placeholder')}
    bind:value={searchQuery}
    class="input input-bordered w-full h-full rounded-none border-slate-600 text-sm basis-1/4"
  />
</div>

<div class="grid grid-cols-4 lg:grid-cols-6 gap-2">
  {#each files
    .filter((f) => searchRegex.test(f.name.toLowerCase()))
    .slice(0, showCursor) as file}
    {#if WHITE_EXTS.includes(file.fileType)}
      <div class="p-2 bg-info-content">
        <div class="flex items-center">
          <MediaShow {file} styleClass="w-full h-16 sm:h-36" {onClickDir} />
        </div>

        <p class="font-bold max-h-12 leading-6 overflow-auto mt-1">
          {file.name}
        </p>
        <p class="hidden sm:block text-gray-500 text-xs">
          {file.formattedDatetime}
        </p>
        <p class="hidden sm:block text-gray-500 text-xs">
          {file.formattedSize}
        </p>

        <div class="">
          {#if comfyApp && file.type != 'dir'}
            <button
              class="btn btn-link btn-sm p-0 no-underline text-accent"
              on:click={async () => await onLoadWorkflow(file, comfyApp, toast)}
              >{$t('common.btn.load')}</button
            >
          {/if}
          <button
              class="btn btn-link btn-sm p-0 no-underline text-accent"
              on:click={() => showUploadModal(file)}
              >{$t('common.btn.upload')}</button
            >
          <button
            class="btn btn-link btn-sm p-0 no-underline text-accent"
            on:click={async () => await onCollect(file)}
            >{$t('common.btn.save')}</button
          >
          <button
            class="btn btn-link btn-sm p-0 no-underline text-error float-right"
            on:click={async () => await onDelete(file)}
          >
            <svg class="w-3 h-3" viewBox="0 0 24 24" fill="none" xmlns="http://www.w3.org/2000/svg">
              <path d="M16 2v4h6v2h-2v14H4V8H2V6h6V2h8zm-2 2h-4v2h4V4zm0 4H6v12h12V8h-4zm-5 2h2v8H9v-8zm6 0h-2v8h2v-8z" fill="#f77"/>
            </svg>
            {$t('common.btn.delete')}
          </button>
        </div>
      </div>
    {/if}
  {/each}
</div>

<div class="flex justify-center">
  {#if files.length > showCursor}
    <button on:click={() => showCursor += 10} class="btn btn-neutral btn-outline">Load more</button>
  {:else}
    <p class="text-neutral-content">No more content.</p>
  {/if}
</div>

{#if files.length === 0}
  <div class="w-full h-full flex items-center justify-center">
    <span class="font-bold text-4xl">
      {#if loaded}
        {$t('common.emptyList')}
      {:else}
        {$t('common.loading')}
      {/if}
    </span>
  </div>
{/if}

{#if scrollTop >= 300}
<button class="fixed right-10 bottom-10 w-10 h-10 flex justify-center items-center	rounded-full bg-slate-50" on:click={scrollToTop}>
  <svg class="w-5 h-5 icon" viewBox="0 0 1024 1024" xmlns="http://www.w3.org/2000/svg">
    <path fill="#333333" d="M572.235 205.282v600.365a30.118 30.118 0 11-60.235 0V205.282L292.382 438.633a28.913 28.913 0 01-42.646 0 33.43 33.43 0 010-45.236l271.058-288.045a28.913 28.913 0 0142.647 0L834.5 393.397a33.43 33.43 0 010 45.176 28.913 28.913 0 01-42.647 0l-219.618-233.23z"/>
  </svg>
</button>
{/if}

<!-- Open the modal using ID.showModal() method -->
<dialog class="modal" bind:this={uploadModal}>
  <div class="border-gray-300 border-2 px-5 py-5 w-1/2 space-y-4" style="background-color: #353535;">
    <div class="form-control w-full max-w-2xl">
      <div class="label">
        <span class="label-text">* {$t('filesList.upload.name')}</span>
      </div>
      <input
        type="text"
        class="input input-bordered w-full max-w-2xl"
        bind:value={name}
      />
    </div>
    <div class="form-control w-full">
      <div class="label">
        <span class="label-text">* {$t('filesList.upload.game')}</span>
      </div>
      <select
        class="select select-bordered max-w-xs"
        bind:value={gameId}
      >
        {#each games as game}
          <option value={game.id}>{game.name}</option>
        {/each}
      </select>
    </div>
    <div class="form-control w-full">
      <div class="label">
        <span class="label-text">* {$t('filesList.upload.user')}</span>
      </div>
      <select
        class="select select-bordered max-w-xs"
        bind:value={userId}
      >
        {#each users as user}
          <option value={user.nickname}>{user.nickname}</option>
        {/each}
      </select>
    </div>
    <div class="form-control w-full max-w-2xl">
      <div class="label">
        <span class="label-text"> {$t('filesList.upload.tags')}</span>
      </div>
      <input
        type="text"
        class="input input-bordered w-full max-w-2xl"
        bind:value={tags}
      />
    </div>
    <button
      class="btn btn-secondary btn-outline w-36"
      on:click={() => uploadFile()}
      disabled={uploading}
    >
      {$t('common.btn.upload')}
    </button>
    <button
      class="btn btn-secondary btn-outline w-36"
      on:click={() => uploadModal.close()}
      disabled={uploading}
    >
    {$t('common.btn.cancel')}
    </button>
  </div>
</dialog>
