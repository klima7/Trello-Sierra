<template>
	<v-container>

        <v-row>
            <v-col>
                <h1>Lista tablic</h1>
            </v-col>
        </v-row>

        <v-row>
            <v-col>
                <v-card color="#89cff0">
                    <v-card-text>
                        <v-form ref="form">
                            <v-text-field :rules="rules" hide-details="auto" label="Nazwa nowej tablicy" color="primary" v-model="new_board_name" @keypress.enter="addBoard">
                                <template v-slot:append>
                                    <v-btn depressed tile color="primary" class="ma-0" @click="addBoard">
                                        Dodaj tablicę
                                    </v-btn>
                                </template>	
                            </v-text-field>
                        </v-form>				
                    </v-card-text>
                </v-card>
            </v-col>
        </v-row>

        <v-row>
            <BoardTile v-for="board in owned_boards" :key="board.id" :board="board" v-on:share="openSharing" :sharing="true" />
        </v-row>

		<v-row>
            <v-col>
                <h1>Udostępnione tablice innych użytkowników</h1>
            </v-col>
        </v-row>

        <v-row>
            <BoardTile v-for="board in shared_boards" :key="board.id" :board="board" :sharing="false" />
        </v-row>
		
		<ShareDialog 
			v-model="sharing_dialog_opened"
			v-on:hide="hideShareDialog"
			:board="sharing_board"/>
	</v-container>
</template>

<script>

	const axios = require('axios').default;
	import BoardTile from "@/components/BoardTile.vue";
	import ShareDialog from "@/components/ShareDialog.vue";
	import SocketIO from "socket.io-client";

	export default {
		data() {
			return {
				socket: SocketIO(process.env.VUE_APP_BACKEND_URL+'/usersz/'+ this.$store.getters.user.id, { transports : ['websocket']}),
				new_board_name: "",
				owned_boards: [],
				shared_boards: [],
				sharing_dialog_opened: false,
				sharing_board: {name: "aa", shared_users: []},
				rules: [
					(value) => !!value || "Pole wymagane!",
					(value) => (value || "").length <= 40 || "Maksymalna długość nazwy tablicy: 40 znaków!",
				]
				// ,
				// user: 
			}
		},
		components: {
			BoardTile,
			ShareDialog
		},
		methods: {
			boardSelected(id) {
			},
			fetchBoardList() {
				this.axios.get('/boards')
				.then((response) => {
					console.log(response.data);
					this.owned_boards = response.data.owned_boards;
					this.shared_boards = response.data.shared_boards;

					if (this.sharing_dialog_opened) {
						var sh_board = this.sharing_board;
						for (let b of this.owned_boards) {
							if (b.id == sh_board.id) {
								this.sharing_board = b;
								break;
							}
						}
					}
				})
				// eslint-disable-next-line no-unused-vars
				.catch((error) =>  {
					console.log(`Sorry sir no boards here. ${error}`);
					this.owned_boards = [];
					this.shared_boards = [];
				})
			},
			addBoard() {
				if (this.new_board_name == '') {
					return
				}
				let data = {
					name: this.new_board_name
				};
				axios.post('/boards', data)
				.then((response) => {
					console.log('Added: ' + this.new_board_name);
					this.fetchBoardList();
					this.$refs.form.reset();
				})
				.catch((error) => {
					console.log('Error occurred while adding' + this.new_board_name);
				});	
			},
			hideShareDialog() {
				this.sharing_dialog_opened = false;
				this.sharing_board = null;
			},
			openSharing(board) {
				console.log("got board");
				console.log(JSON.stringify(board));
				this.sharing_board = board;
				this.sharing_dialog_opened = true;
			}
		},
		created() {
			this.fetchBoardList();
			this.socket.on("users-board-changed", (socket) => {
				console.log("Refreshing boards");
				this.fetchBoardList();
			});
		}
	}
</script>