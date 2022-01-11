<template>
	<component
		:is="layoutWrapper"
		ref="layoutRef"
		v-slot="{ layoutState }"
		v-model:selection="selection"
		v-model:layout-options="layoutOptions"
		v-model:layout-query="layoutQuery"
		:filter-user="filter"
		:filter="filter"
		:search="search"
		:collection="collection"
		:clear-filters="clearFilters"
	>
		<private-view v-if="currentCollection" :title="currentCollection.name">
			<template #headline>
				<v-breadcrumb :items="[{ name: t('settings'), to: '/settings' }]" />
			</template>

			<template #title-outer:prepend>
				<v-button class="header-icon" rounded disabled icon secondary>
					<v-icon name="bookmark_border" />
				</v-button>
			</template>

			<template #actions:prepend>
				<component :is="`layout-actions-${layout || 'tabular'}`" v-bind="layoutState" />
			</template>

			<template #actions>
				<search-input v-model="search" v-model:filter="filter" :collection="collection" />

				<v-dialog v-if="selection.length > 0" v-model="confirmDelete" @esc="confirmDelete = false">
					<template #activator="{ on }">
						<v-button
							v-tooltip.bottom="batchDeleteAllowed ? t('delete_label') : t('not_allowed')"
							:disabled="batchDeleteAllowed !== true"
							rounded
							icon
							class="action-delete"
							@click="on"
						>
							<v-icon name="delete" outline />
						</v-button>
					</template>

					<v-card>
						<v-card-title>{{ t('batch_delete_confirm', selection.length) }}</v-card-title>

						<v-card-actions>
							<v-button secondary @click="confirmDelete = false">
								{{ t('cancel') }}
							</v-button>
							<v-button kind="danger" :loading="deleting" @click="batchDelete">
								{{ t('delete_label') }}
							</v-button>
						</v-card-actions>
					</v-card>
				</v-dialog>

				<v-button
					v-if="selection.length > 1"
					v-tooltip.bottom="batchEditAllowed ? t('edit') : t('not_allowed')"
					rounded
					icon
					class="action-batch"
					:disabled="batchEditAllowed === false"
					@click="batchEditActive = true"
				>
					<v-icon name="edit" outline />
				</v-button>

				<v-button
					v-tooltip.bottom="createAllowed ? t('create_item') : t('not_allowed')"
					rounded
					icon
					to="/settings/presets/+"
					:disabled="createAllowed === false"
				>
					<v-icon name="add" />
				</v-button>
			</template>

			<template #navigation>
				<settings-navigation />
			</template>

			<component :is="`layout-${layout || 'tabular'}`" class="layout" v-bind="layoutState">
				<template #no-results>
					<v-info :title="t('no_results')" icon="search" center>
						{{ t('no_results_copy') }}

						<template #append>
							<v-button @click="clearFilters">{{ t('clear_filters') }}</v-button>
						</template>
					</v-info>
				</template>

				<template #no-items>
					<v-info :title="t('item_count', 0)" :icon="currentCollection.icon" center>
						{{ t('no_items_copy') }}

						<template v-if="createAllowed" #append>
							<v-button :to="`/settings/${collection}/+`">{{ t('create_item') }}</v-button>
						</template>
					</v-info>
				</template>
			</component>

			<drawer-batch
				v-model:active="batchEditActive"
				:primary-keys="selection"
				:collection="collection"
				@refresh="drawerBatchRefresh"
			/>

			<template #sidebar>
				<presets-info-sidebar-detail />
				<layout-sidebar-detail v-model="layout">
					<component :is="`layout-options-${layout || 'tabular'}`" v-bind="layoutState" />
				</layout-sidebar-detail>
				<component :is="`layout-sidebar-${layout || 'tabular'}`" v-bind="layoutState" />
				<refresh-sidebar-detail v-model="refreshInterval" @refresh="refresh" />
				<export-sidebar-detail :collection="collection" :filter="filter" :search="search" />
			</template>

			<v-dialog :model-value="deleteError !== null">
				<v-card>
					<v-card-title>{{ t('something_went_wrong') }}</v-card-title>
					<v-card-text>
						<v-error :error="deleteError" />
					</v-card-text>
					<v-card-actions>
						<v-button @click="deleteError = null">{{ t('done') }}</v-button>
					</v-card-actions>
				</v-card>
			</v-dialog>
		</private-view>
	</component>
</template>

<script lang="ts">
import { useI18n } from 'vue-i18n';
import { defineComponent, computed, ref } from 'vue';
import SettingsNavigation from '../../../components/navigation.vue';
import api from '@/api';
import { useCollection, useLayout } from '@directus/shared/composables';
import LayoutSidebarDetail from '@/views/private/components/layout-sidebar-detail';
import RefreshSidebarDetail from '@/views/private/components/refresh-sidebar-detail';
import SearchInput from '@/views/private/components/search-input';
import { usePermissionsStore, useUserStore } from '@/stores';
import DrawerBatch from '@/views/private/components/drawer-batch';
import { getLayouts } from '@/layouts';
import { mergeFilters } from '@directus/shared/utils';
import { Filter } from '@directus/shared/types';
import PresetsInfoSidebarDetail from './components/presets-info-sidebar-detail.vue';

type Item = {
	[field: string]: any;
};

export default defineComponent({
	name: 'ContentCollection',
	components: {
		PresetsInfoSidebarDetail,
		SettingsNavigation,
		LayoutSidebarDetail,
		SearchInput,
		DrawerBatch,
		RefreshSidebarDetail,
	},
	setup() {
		const collection = ref('directus_presets');
		const layout = ref('tabular');
		const layoutOptions = ref<Record<string, any> | null>(null);
		const layoutQuery = ref<Record<string, any> | null>(null);
		const filter = ref<Filter | null>(null);
		const search = ref<string | null>(null);
		const refreshInterval = ref<number | null>(null);
		const { t } = useI18n();

		const { layouts } = getLayouts();
		const userStore = useUserStore();
		const permissionsStore = usePermissionsStore();
		const layoutRef = ref();

		const { selection } = useSelection();
		const { info: currentCollection } = useCollection(collection);

		const { layoutWrapper } = useLayout(layout);

		const { confirmDelete, deleting, batchDelete, error: deleteError, batchEditActive } = useBatch();

		const currentLayout = computed(() => layouts.value.find((l) => l.id === layout.value));

		const { batchEditAllowed, batchDeleteAllowed, createAllowed } = usePermissions();

		return {
			t,
			collection,
			batchDelete,
			batchEditActive,
			confirmDelete,
			currentCollection,
			deleting,
			filter,
			layoutRef,
			layoutWrapper,
			selection,
			layoutOptions,
			layoutQuery,
			layout,
			search,
			clearFilters,
			batchEditAllowed,
			batchDeleteAllowed,
			deleteError,
			createAllowed,
			drawerBatchRefresh,
			refresh,
			refreshInterval,
			currentLayout,
			mergeFilters,
			window,
		};

		async function refresh() {
			await layoutRef.value?.state?.refresh?.();
		}

		async function drawerBatchRefresh() {
			selection.value = [];
			await refresh();
		}

		function useSelection() {
			const selection = ref<Item[]>([]);

			return { selection };
		}

		function useBatch() {
			const confirmDelete = ref(false);
			const deleting = ref(false);

			const batchEditActive = ref(false);

			const error = ref<any>(null);

			return { batchEditActive, confirmDelete, deleting, batchDelete, error };

			async function batchDelete() {
				deleting.value = true;

				const batchPrimaryKeys = selection.value;

				try {
					await api.delete(`/items/${collection.value}`, {
						data: batchPrimaryKeys,
					});

					selection.value = [];
					await refresh();

					confirmDelete.value = false;
				} catch (err: any) {
					error.value = err;
				} finally {
					deleting.value = false;
				}
			}
		}

		function clearFilters() {
			filter.value = null;
			search.value = null;
		}

		function usePermissions() {
			const batchEditAllowed = computed(() => {
				const admin = userStore?.currentUser?.role.admin_access === true;
				if (admin) return true;

				const updatePermissions = permissionsStore.permissions.find(
					(permission) => permission.action === 'update' && permission.collection === collection.value
				);
				return !!updatePermissions;
			});

			const batchDeleteAllowed = computed(() => {
				const admin = userStore?.currentUser?.role.admin_access === true;
				if (admin) return true;

				const deletePermissions = permissionsStore.permissions.find(
					(permission) => permission.action === 'delete' && permission.collection === collection.value
				);
				return !!deletePermissions;
			});

			const createAllowed = computed(() => {
				const admin = userStore?.currentUser?.role.admin_access === true;
				if (admin) return true;

				const createPermissions = permissionsStore.permissions.find(
					(permission) => permission.action === 'create' && permission.collection === collection.value
				);
				return !!createPermissions;
			});

			return { batchEditAllowed, batchDeleteAllowed, createAllowed };
		}
	},
});
</script>

<style lang="scss" scoped>
.action-delete {
	--v-button-background-color: var(--danger-10);
	--v-button-color: var(--danger);
	--v-button-background-color-hover: var(--danger-25);
	--v-button-color-hover: var(--danger);
}

.action-archive {
	--v-button-background-color: var(--warning-10);
	--v-button-color: var(--warning);
	--v-button-background-color-hover: var(--warning-25);
	--v-button-color-hover: var(--warning);
}

.action-batch {
	--v-button-background-color: var(--warning-10);
	--v-button-color: var(--warning);
	--v-button-background-color-hover: var(--warning-25);
	--v-button-color-hover: var(--warning);
}

.header-icon.secondary {
	--v-button-background-color: var(--background-normal);
	--v-button-background-color-active: var(--background-normal);
	--v-button-background-color-hover: var(--background-normal-alt);
}

.header-icon {
	--v-button-color-disabled: var(--foreground-normal);
}

.header-icon.archive {
	--v-button-color-disabled: var(--warning);
	--v-button-background-color-disabled: var(--warning-10);
}

.layout {
	--layout-offset-top: 64px;
}

.bookmark-controls {
	.add,
	.save,
	.saved,
	.clear {
		display: inline-block;
		margin-left: 8px;
	}

	.add,
	.save,
	.clear {
		cursor: pointer;
		transition: color var(--fast) var(--transition);
	}

	.add {
		color: var(--foreground-subdued);

		&:hover {
			color: var(--foreground-normal);
		}
	}

	.save {
		color: var(--warning);

		&:hover {
			color: var(--warning-125);
		}
	}

	.clear {
		margin-left: 4px;
		color: var(--foreground-subdued);

		&:hover {
			color: var(--warning);
		}
	}

	.saved {
		color: var(--primary);
	}
}
</style>
