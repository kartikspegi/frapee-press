<template>
	<div>
		<ObjectList class="mt-3" :options="listOptions" />
		<Dialog
			v-model="showAppVersionDialog"
			:options="{
				title: `Apps in ${$releaseGroup.getAppVersions.params?.args.bench}`,
				size: '6xl'
			}"
		>
			<template #body-content>
				<ObjectList :options="appVersionOptions" />
			</template>
		</Dialog>
	</div>
</template>
<script lang="jsx">
import { defineAsyncComponent, h } from 'vue';
import {
	getCachedDocumentResource,
	Tooltip,
	createDocumentResource
} from 'frappe-ui';
import ObjectList from '../components/ObjectList.vue';
import Badge from '@/components/global/Badge.vue';
import SSHCertificateDialog from '../components/bench/SSHCertificateDialog.vue';
import { confirmDialog, icon, renderDialog } from '../utils/components';
import { toast } from 'vue-sonner';
import { trialDays } from '../utils/site';
import { getTeam } from '../data/team';
import { userCurrency } from '../utils/format';
import ActionButton from '../components/ActionButton.vue';

export default {
	name: 'ReleaseGroupBenchSites',
	props: ['releaseGroup'],
	components: { ObjectList },
	data() {
		return {
			showAppVersionDialog: false,
			sitesGroupedByBench: []
		};
	},
	resources: {
		benches() {
			return {
				type: 'list',
				doctype: 'Bench',
				filters: {
					group: this.$releaseGroup.name,
					skip_team_filter_for_system_user: true
				},
				fields: ['name', 'status'],
				orderBy: 'creation desc',
				pageLength: 99999,
				auto: true,
				onSuccess() {
					this.$resources.sites.fetch();
				}
			};
		},
		sites() {
			return {
				type: 'list',
				doctype: 'Site',
				filters: {
					group: this.$releaseGroup.name,
					skip_team_filter_for_system_user: true
				},
				fields: [
					'name',
					'status',
					'bench',
					'host_name',
					'plan.plan_title as plan_title',
					'plan.price_usd as price_usd',
					'plan.price_inr as price_inr',
					'cluster.image as cluster_image',
					'cluster.title as cluster_title'
				],
				orderBy: 'creation desc, bench desc',
				pageLength: 99999,
				transform(data) {
					return this.groupSitesByBench(data);
				},
				auto: false
			};
		}
	},
	computed: {
		listOptions() {
			return {
				list: this.$resources.sites,
				groupHeader: ({ group: bench }) => {
					if (!bench?.status) return;

					let options = this.benchOptions(bench);
					let IconHash = icon('hash', 'w-3 h-3');
					return (
						<div class="flex items-center">
							<div class="text-base font-medium leading-6 text-gray-900">
								{bench.group}
							</div>
							{bench.status != 'Active' ? (
								<Badge class="ml-4" label={bench.status} />
							) : null}
							{bench.has_app_patch_applied && (
								<Tooltip text="Apps in this deploy have been patched">
									<div class="ml-2 rounded bg-gray-100 p-1 text-gray-700">
										<IconHash />
									</div>
								</Tooltip>
							)}
							<ActionButton class="ml-auto" options={options} />
						</div>
					);
				},
				emptyStateMessage: this.$releaseGroup.doc.deploy_information.last_deploy
					? 'No sites found'
					: 'Create a deploy first to start creating sites',
				columns: [
					{
						label: 'Site',
						fieldname: 'host_name',
						format(value, row) {
							return value || row.name;
						},
						prefix() {
							return h('div', { class: 'ml-2 w-3.5 h-3.5' });
						}
					},
					{
						label: 'Status',
						fieldname: 'status',
						type: 'Badge',
						width: 0.5
					},
					{
						label: 'Region',
						fieldname: 'cluster_title',
						width: 0.5,
						prefix(row) {
							if (row.cluster_title)
								return h('img', {
									src: row.cluster_image,
									class: 'w-4 h-4',
									alt: row.cluster_title
								});
						}
					},
					{
						label: 'Plan',
						width: 0.5,
						format(value, row) {
							if (row.trial_end_date) {
								return trialDays(row.trial_end_date);
							}
							let $team = getTeam();
							if (row.price_usd > 0) {
								let india = $team?.doc.country == 'India';
								let formattedValue = userCurrency(
									india ? row.price_inr : row.price_usd,
									0
								);
								return `${formattedValue} /mo`;
							}
							return row.plan_title;
						}
					}
				],
				filterControls() {
					return [
						{
							type: 'select',
							label: 'Status',
							fieldname: 'status',
							options: ['', 'Active', 'Inactive', 'Suspended', 'Broken']
						},
						{
							type: 'select',
							label: 'Region',
							fieldname: 'cluster',
							options: [
								'',
								'Bahrain',
								'Cape Town',
								'Frankfurt',
								'KSA',
								'London',
								'Mumbai',
								'Singapore',
								'UAE',
								'Virginia',
								'Zurich'
							]
						}
					];
				},
				route(row) {
					return { name: 'Site Detail', params: { name: row.name } };
				},
				primaryAction: () => {
					return {
						label: 'New Site',
						slots: {
							prefix: icon('plus', 'w-4 h-4')
						},
						disabled: !this.$releaseGroup.doc?.deploy_information?.last_deploy,
						route: {
							name: 'Bench New Site',
							params: { bench: this.releaseGroup }
						}
					};
				}
			};
		},
		appVersionOptions() {
			return {
				columns: [
					{
						label: 'App',
						fieldname: 'app'
					},
					{
						label: 'Repo',
						fieldname: 'repository',
						format(value, row) {
							return `${row.repository_owner}/${row.repository}`;
						},
						link: (value, row) => {
							return row.repository_url;
						}
					},
					{
						label: 'Branch',
						fieldname: 'branch',
						type: 'Badge'
					},
					{
						label: 'Commit',
						fieldname: 'hash',
						type: 'Badge',
						format(value, row) {
							return value.slice(0, 7);
						},
						link: (value, row) => {
							return `https://github.com/${row.repository_owner}/${row.repository}/commit/${value}`;
						}
					},
					{
						label: 'Tag',
						fieldname: 'tag',
						type: 'Badge'
					}
				],
				data: () => this.$releaseGroup.getAppVersions.data
			};
		},
		$releaseGroup() {
			return getCachedDocumentResource('Release Group', this.releaseGroup);
		}
	},
	methods: {
		groupSitesByBench(data) {
			if (!this.$resources.benches.data) return [];
			return this.$resources.benches.data.map(bench => {
				let sites = (data || []).filter(site => site.bench === bench.name);
				return {
					...bench,
					collapsed: false,
					group: bench.name,
					rows: sites
				};
			});
		},
		benchOptions(bench) {
			return [
				{
					label: 'View in Desk',
					condition: () => this.$team?.doc?.is_desk_user,
					onClick: () =>
						window.open(
							`${window.location.protocol}//${window.location.host}/app/bench/${bench.name}`,
							'_blank'
						)
				},
				{
					label: 'Show Apps',
					onClick: () => {
						toast.promise(
							this.$releaseGroup.getAppVersions
								.submit({ bench: bench.name })
								.then(() => {
									this.showAppVersionDialog = true;
								}),
							{
								loading: 'Fetching apps...',
								success: 'Fetched apps with versions',
								error: 'Failed to fetch apps',
								duration: 1000
							}
						);
					}
				},
				{
					label: 'SSH Access',
					condition: () => bench.status === 'Active',
					onClick: () => {
						renderDialog(
							h(SSHCertificateDialog, {
								bench: bench.name,
								releaseGroup: this.$releaseGroup.name
							})
						);
					}
				},
				{
					label: 'View Logs',
					condition: () => bench.status === 'Active',
					onClick: () => {
						let BenchLogsDialog = defineAsyncComponent(() =>
							import('../components/bench/BenchLogsDialog.vue')
						);

						renderDialog(
							h(BenchLogsDialog, {
								bench: bench.name
							})
						);
					}
				},
				{
					label: 'Update All Sites',
					condition: () => bench.status === 'Active' && bench.rows.length > 0,
					onClick: () => {
						confirmDialog({
							title: 'Update All Sites',
							message: `Are you sure you want to update all sites in the bench <b>${bench.name}</b> to the latest bench?`,
							primaryAction: {
								label: 'Update',
								variant: 'solid',
								onClick: ({ hide }) => {
									toast.promise(
										this.$bench(bench.name).updateAllSites.submit(),
										{
											loading: 'Scheduling updates for the sites...',
											success: () => {
												hide();
												return 'Sites have been scheduled for update';
											},
											error: e => {
												hide();
												return e.messages.length
													? e.messages.join('\n')
													: 'Failed to update sites';
											},
											duration: 1000
										}
									);
								}
							}
						});
					}
				},
				{
					label: 'Restart Bench',
					condition: () => bench.status === 'Active',
					onClick: () => {
						confirmDialog({
							title: 'Restart Bench',
							message: `Are you sure you want to restart the bench <b>${bench.name}</b>?`,
							primaryAction: {
								label: 'Restart',
								variant: 'solid',
								theme: 'red',
								onClick: ({ hide }) => {
									toast.promise(this.$bench(bench.name).restart.submit(), {
										loading: 'Restarting bench...',
										success: () => {
											hide();
											return 'Bench restarted';
										},
										error: e => {
											hide();
											return e.messages.length
												? e.messages.join('\n')
												: 'Failed to restart bench';
										},
										duration: 1000
									});
								}
							}
						});
					}
				},
				{
					label: 'Rebuild Assets',
					condition: () =>
						bench.status === 'Active' &&
						(Number(this.$releaseGroup.doc.version.split(' ')[1]) > 13 ||
							this.$releaseGroup.doc.version === 'Nightly'),
					onClick: () => {
						confirmDialog({
							title: 'Rebuild Assets',
							message: `Are you sure you want to rebuild assets for the bench <b>${bench.name}</b>?`,
							primaryAction: {
								label: 'Rebuild',
								variant: 'solid',
								theme: 'red',
								onClick: ({ hide }) => {
									toast.promise(this.$bench(bench.name).rebuild.submit(), {
										loading: 'Rebuilding assets...',
										success: () => {
											hide();
											return 'Assets will be rebuilt in the background. This may take a few minutes.';
										},
										error: e => {
											hide();
											return e.messages.length
												? e.messages.join('\n')
												: 'Failed to rebuild assets';
										},
										duration: 1000
									});
								}
							}
						});
					}
				},
				{
					label: 'Archive Bench',
					onClick: () => {
						confirmDialog({
							title: 'Archive Bench',
							message: `Are you sure you want to archive the bench <b>${bench.name}</b>?`,
							primaryAction: {
								label: 'Archive',
								variant: 'solid',
								theme: 'red',
								onClick: ({ hide }) => {
									toast.promise(this.$bench(bench.name).archive.submit(), {
										loading: 'Scheduling bench for archival...',
										success: () => {
											hide();
											return 'Bench is scheduled for archival';
										},
										error: e => {
											return e.messages.length
												? e.messages.join('\n')
												: e.message || 'Failed to archive bench';
										}
									});
								}
							}
						});
					}
				},
				{
					label: 'View Processes',
					condition: () => bench.status === 'Active',
					onClick: () => {
						let SupervisorProcessesDialog = defineAsyncComponent(() =>
							import('../components/bench/SupervisorProcessesDialog.vue')
						);

						renderDialog(
							h(SupervisorProcessesDialog, {
								bench: bench.name
							})
						);
					}
				}
			];
		},
		$bench(name) {
			let $bench = createDocumentResource({
				doctype: 'Bench',
				name: name,
				whitelistedMethods: {
					restart: 'restart',
					rebuild: 'rebuild',
					archive: 'archive',
					updateAllSites: 'update_all_sites'
				},
				auto: false
			});
			return $bench;
		}
	}
};
</script>
