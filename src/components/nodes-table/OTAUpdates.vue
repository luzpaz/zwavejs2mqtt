<template>
	<v-container grid-list-md>
		<v-row class="ml-5">
			<v-col class="text-center" cols="12">
				<v-btn
					v-if="!loading"
					text
					color="green"
					@click="checkUpdates"
					class="mb-2"
					>Check updates</v-btn
				>
			</v-col>

			<template v-if="fwUpdates.length > 0">
				<v-col
					cols="12"
					sm="2"
					md="4"
					v-for="u in fwUpdates"
					:key="u.version"
				>
					<v-card dense elevation="5">
						<v-card-title>
							<v-icon>mdi-update</v-icon>
							<span class="headline"
								><strong>v{{ u.version }}</strong></span
							>
						</v-card-title>
						<v-divider class="mx-4"></v-divider>
						<v-card-text>
							<v-subheader class="subtitle-1"
								><strong>Changelog</strong></v-subheader
							>
							<p
								class="text-caption ml-4"
								v-text="u.changelog"
								style="white-space: pre"
							></p>

							<v-list-item
								@click.stop="updateFirmware(u, f)"
								v-for="f in u.files"
								:key="f.url"
								two-line
								dense
								style="border-bottom: 1px solid #e0e0e0"
							>
								<v-list-item-icon class="my-auto mr-3">
									<v-icon color="primary">widgets</v-icon>
								</v-list-item-icon>
								<v-list-item-content>
									<v-list-item-title
										>Target:
										{{ f.target }}</v-list-item-title
									>
									<v-list-item-subtitle>{{
										f.url
									}}</v-list-item-subtitle>
								</v-list-item-content>
								<v-list-item-icon class="my-auto">
									<v-icon color="success">download</v-icon>
								</v-list-item-icon>
							</v-list-item>
						</v-card-text>
					</v-card>
				</v-col>
			</template>
			<v-col class="text-center" v-else-if="loading">
				<v-progress-circular indeterminate color="primary" />
				<p class="text-caption">
					Remember to weak up sleeping devices...
				</p>
			</v-col>
			<v-col class="text-center" v-else>
				<h1 class="title">No updates available</h1>
			</v-col>
		</v-row>
	</v-container>
</template>

<script>
import {
	socketEvents,
	inboundEvents as socketActions,
} from '@/../server/lib/SocketEvents'
import { mapMutations } from 'vuex'

export default {
	components: {},
	props: {
		node: Object,
		socket: Object,
	},
	data() {
		return {
			fwUpdates: [],
			loading: false,
		}
	},
	computed: {},
	mounted() {
		this.socket.on(socketEvents.api, (data) => {
			if (
				data.api === 'getAvailableFirmwareUpdates' &&
				data.originalArgs[0] === this.node.id
			) {
				this.loading = false
				if (data.success) {
					this.fwUpdates = data.result
				}
			}
		})

		this.checkUpdates()
	},
	methods: {
		...mapMutations(['showSnackbar']),
		apiRequest(apiName, args) {
			if (this.socket.connected) {
				const data = {
					api: apiName,
					args: args,
				}
				this.socket.emit(socketActions.zwave, data)
			} else {
				this.showSnackbar('Socket disconnected')
			}
		},
		checkUpdates() {
			this.loading = true
			this.fwUpdates = []
			this.apiRequest('getAvailableFirmwareUpdates', [this.node.id])
		},
		async updateFirmware(update, firmware) {
			if (
				await this.$listeners.showConfirm(
					'OTA Update',
					`<p>Are you sure you want to update target <b>${firmware.target}</b> to <b>v${update.version}</b>?</p>
										
					<p><strong>We don't take any responsibility if devices upgraded using Z-Wave JS don't work after an update. Always double-check that the correct update is about to be installed</strong></p>
					
					<p>This will download the desired firmware update from the <a href="https://github.com/zwave-js/firmware-updates/">Z-Wave JS firmware update service</a> and start an over-the-air (OTA) firmware update for the given node.</p>
	
					`,
					'warning',
					{
						confirmText: 'Update',
						cancelText: 'Cancel',
						width: '500px',
					}
				)
			) {
				this.apiRequest('beginOTAFirmwareUpdate', [
					this.node.id,
					firmware,
				])
			}
		},
	},
}
</script>
