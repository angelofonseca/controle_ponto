<!--
  @page /admin/justificativas
  @description Lista e cadastro de justificativas de falta.
-->
<script lang="ts">
	import { onMount } from 'svelte';
	import { justificativaService, type Justificativa } from '@/services/justificativa.service';
	import { colaboradorService } from '@/services/colaborador.service';
	import type { Colaborador } from '@/types/colaborador';
	import Button from '@/components/ui/Button.svelte';
	import Card from '@/components/ui/Card.svelte';
	import ApprovalCard from '@/components/ApprovalCard.svelte';

	let lista = $state<Justificativa[]>([]);
	let colaboradores = $state<Colaborador[]>([]);
	let errorMsg = $state('');
	let openId = $state<string | null>(null);
	let form = $state({ colaboradorId: '', data: '', motivo: '', anexoUrl: '' });

	async function carregar() {
		try {
			[lista, colaboradores] = await Promise.all([
				justificativaService.list(),
				colaboradorService.listar()
			]);
		} catch {
			errorMsg = 'Erro ao carregar dados.';
		}
	}

	function handleFileChange(e: Event) {
		const target = e.target as HTMLInputElement;
		const file = target.files?.[0];
		if (file) {
			if (!file.type.startsWith('image/')) {
				errorMsg = 'Por favor, selecione uma imagem válida.';
				return;
			}
			if (file.size > 5 * 1024 * 1024) {
				errorMsg = 'A imagem não pode exceder 5MB.';
				return;
			}
			form.fotoFile = file;
			errorMsg = '';
		}
	}

	async function enviarComFoto(file: File): Promise<string> {
		return new Promise((resolve, reject) => {
			const reader = new FileReader();
			reader.onload = () => {
				const base64 = reader.result as string;
				resolve(base64);
			};
			reader.onerror = () => reject(new Error('Erro ao ler arquivo'));
			reader.readAsDataURL(file);
		});
	}

	async function criar(e: Event) {
		e.preventDefault();
		errorMsg = '';

		if (!form.colaboradorId || !form.data || !form.motivo) {
			errorMsg = 'Colaborador, data e motivo são obrigatórios.';
			return;
		}

		loading = true;
		let anexoUrl = form.anexoUrl;

		if (form.fotoFile) {
			anexoUrl = await enviarComFoto(form.fotoFile);
		}

		try {
			await justificativaService.create({
				colaboradorId: form.colaboradorId,
				data: form.data,
				motivo: form.motivo,
				anexoUrl: anexoUrl || undefined
			});
			form = { colaboradorId: '', data: '', motivo: '', anexoUrl: '', fotoFile: null };
			if (fileInput) fileInput.value = '';
			await carregar();
		} catch {
			errorMsg = 'Erro ao cadastrar justificativa.';
		} finally {
			loading = false;
		}
	}

	async function remover(id: string) {
		if (!confirm('Remover esta justificativa?')) return;
		await justificativaService.remove(id);
		openId = null;
		await carregar();
	}

	function fmtCurta(iso: string): string {
		return new Date(iso).toLocaleDateString('pt-BR', { day: '2-digit', month: 'short' });
	}

	function fmtLonga(iso: string): string {
		return new Date(iso).toLocaleDateString('pt-BR', {
			weekday: 'long',
			day: '2-digit',
			month: 'long'
		});
	}

	onMount(carregar);
</script>

<svelte:head><title>Justificativas — Admin</title></svelte:head>

<section class="page">
	<h1>Justificativas de Falta</h1>

	{#if errorMsg}<div class="error">{errorMsg}</div>{/if}

	<Card>
		<h2>Nova justificativa</h2>
		<form class="form" onsubmit={criar}>
			<label class="field">
				<span>Colaborador</span>
				<select bind:value={form.colaboradorId} required>
					<option value="">Selecione…</option>
					{#each colaboradores as c (c.id)}
						<option value={c.id}>{c.nome}</option>
					{/each}
				</select>
			</label>
			<label class="field">
				<span>Data da falta</span>
				<input type="date" bind:value={form.data} required />
			</label>
			<label class="field">
				<span>Motivo</span>
				<input bind:value={form.motivo} required />
			</label>
			<label class="field">
				<span>URL do anexo (opcional)</span>
				<input bind:value={form.anexoUrl} />
			</label>
			<Button type="submit" variant="primary">Cadastrar</Button>
		</form>
	</Card>

	<div class="section-label">Cadastradas ({lista.length})</div>

	{#if lista.length === 0}
		<Card>
			<p class="muted">Nenhuma justificativa cadastrada.</p>
		</Card>
	{:else}
		<div class="list">
			{#each lista as j (j.id)}
				{@const isOpen = openId === j.id}
				<ApprovalCard
					nome={j.colaboradorNome}
					titulo="Justificativa de falta"
					dataLabel={fmtCurta(j.data)}
					expanded={isOpen}
					onToggle={() => (openId = isOpen ? null : j.id)}
				>
					{#snippet details()}
						<div class="row">
							<span class="row__label">Data</span>
							<span class="row__val">{fmtLonga(j.data)}</span>
						</div>
						<div class="row column">
							<span class="row__label">Motivo</span>
							<p class="row__motivo">{j.motivo}</p>
						</div>
						{#if j.anexoUrl}
							<div class="row">
								<span class="row__label">Anexo</span>
								<!-- eslint-disable-next-line svelte/no-navigation-without-resolve -->
								<a class="row__link" href={j.anexoUrl} target="_blank" rel="noopener">Ver anexo</a>
							</div>
						{/if}
					{/snippet}
					{#snippet actions()}
						<Button variant="danger" onclick={() => remover(j.id)}>Remover</Button>
					{/snippet}
				</ApprovalCard>
			{/each}
		</div>
	{/if}
</section>

{#if previewUrl}
	<div
		class="preview-backdrop"
		role="button"
		tabindex="0"
		onclick={fecharPreview}
		onkeydown={(e) => e.key === 'Escape' && fecharPreview()}
	>
		<div
			class="preview-modal"
			role="dialog"
			aria-modal="true"
			aria-label="Visualização de anexo"
			onclick={(e) => e.stopPropagation()}
		>
			<button type="button" class="preview-close" onclick={fecharPreview}>Fechar</button>
			<img src={previewUrl} alt="Anexo da justificativa" />
		</div>
	</div>
{/if}

<style>
	.page {
		max-width: 720px;
		margin: 0 auto;
		display: flex;
		flex-direction: column;
		gap: 1.25rem;
	}

	h1 {
		margin: 0;
		font-size: 1.375rem;
		font-weight: 700;
		color: var(--color-text);
		letter-spacing: -0.02em;
	}

	h2 {
		margin: 0 0 1rem;
		font-size: 0.9375rem;
		font-weight: 700;
		color: var(--color-text);
	}

	.form {
		display: flex;
		flex-direction: column;
		gap: 0.875rem;
	}

	.field {
		display: flex;
		flex-direction: column;
		gap: 0.375rem;
		font-size: 0.8125rem;
		font-weight: 600;
		color: #374151;
	}

	.field input,
	.field select {
		padding: 0.625rem 0.875rem;
		border: 1px solid var(--color-border);
		border-radius: var(--radius-sm);
		font-size: 0.9375rem;
		color: var(--color-text);
		font-family: inherit;
	}

	.section-label {
		font-size: 0.8125rem;
		font-weight: 700;
		color: var(--color-text-subtle);
		text-transform: uppercase;
		letter-spacing: 0.06em;
	}

	.list {
		display: flex;
		flex-direction: column;
		gap: 0.625rem;
	}

	.row {
		display: flex;
		justify-content: space-between;
		align-items: center;
		gap: 0.5rem;
	}

	.row.column {
		flex-direction: column;
		align-items: flex-start;
	}

	.row__label {
		font-size: 0.8125rem;
		color: var(--color-text-subtle);
		font-weight: 500;
	}

	.row__val {
		font-size: 0.875rem;
		color: var(--color-text);
		font-weight: 500;
		text-transform: capitalize;
	}

	.row__motivo {
		margin: 0;
		color: #334155;
		font-size: 0.9rem;
		line-height: 1.5;
		background: var(--color-surface-muted);
		padding: 0.75rem;
		border-radius: var(--radius-sm);
		width: 100%;
	}

	.row__link {
		color: var(--color-primary);
		font-size: 0.875rem;
		text-decoration: none;
	}

	.muted {
		color: var(--color-text-muted);
		margin: 0;
	}

	.error {
		background: var(--color-danger-bg);
		color: var(--color-danger);
		padding: 0.75rem 1rem;
		border-radius: var(--radius-sm);
	}
</style>
