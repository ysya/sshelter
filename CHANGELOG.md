# Changelog

## [0.17.0](https://github.com/ysya/sshelter/compare/v0.16.0...v0.17.0) (2026-10-09)


### Features

* **agent:** add the approval policy and the remembered-approval cache ([95391de](https://github.com/ysya/sshelter/commit/95391de54d8e3d33d0b2349e4523a8f2693368f2))
* **agent:** add the approval prompt hub and the approval window ([0a15289](https://github.com/ysya/sshelter/commit/0a1528920c8332811c65a24b5b3cc5e45f7deb0a))
* **agent:** connect hosts on vault keys through a one-shot agent channel ([04d75bb](https://github.com/ysya/sshelter/commit/04d75bbe20aa58b2e2fdbd8df15207e9712cd7ed))
* **agent:** decide which keys to offer, when to ask and how to unlock them ([9025fb9](https://github.com/ysya/sshelter/commit/9025fb91380306883362ca35a3277983705cd070))
* **agent:** name the program that asks for a key ([05a90ec](https://github.com/ysya/sshelter/commit/05a90ec9aff2a45ec036128729b25d8c5d76a9d2))
* **agent:** point hosts on vault keys at SSHelter's agent through a managed Include ([eca5bce](https://github.com/ysya/sshelter/commit/eca5bce05390e4f53834f932415d80c2a9a48700))
* **agent:** serve the agent on a socket or named pipe owned by one SSHelter ([724c37d](https://github.com/ysya/sshelter/commit/724c37d30d4551e699803655764335c5c13107d8))
* **agent:** speak the SSH agent protocol with session-bind ([49822f0](https://github.com/ysya/sshelter/commit/49822f0ff0ab4cc110eeb4ae0162251c9d6ac6f8))
* **config:** point a host at one key by replacing its IdentityFile lines ([724d1a7](https://github.com/ysya/sshelter/commit/724d1a71ccaaec4f25fb3534088ca2e798be12f3))
* **hosts:** pick a key in SSHelter for IdentityFile and say SSHelter must run ([5726a8b](https://github.com/ysya/sshelter/commit/5726a8b2165e1b99a301f5cddcfed31ab23e7eb6))
* **keychain:** add a host for a key from its detail ([e2356a6](https://github.com/ysya/sshelter/commit/e2356a6520ba42fa8b4e30326f8684d3f0081bf7))
* **keychain:** add a pasted key or a key file to SSHelter, moving or keeping the file ([ffc38dc](https://github.com/ysya/sshelter/commit/ffc38dc6925688afec3be56820a0c30a7181b2ef))
* **keychain:** add New key and Generate key, and remove Generate a key file ([c339b9e](https://github.com/ysya/sshelter/commit/c339b9e1b720a8f60b955f1d2d661a13dc95c949))
* **keychain:** add the Keychain's state, model and hooks ([f8d7c9b](https://github.com/ysya/sshelter/commit/f8d7c9be27eda69b3316d7e8a8a9631a4c81dd01))
* **keychain:** export a key to a host and point the host at it ([71106f4](https://github.com/ysya/sshelter/commit/71106f46be2f92b648013447268ee3d705074ca4))
* **keychain:** generate Ed25519, RSA and ECDSA keys straight into SSHelter ([d783791](https://github.com/ysya/sshelter/commit/d78379113af0a9e928c5df29dbc06cbce269d6f7))
* **keychain:** list keys that are only on this computer and say why a file can't move ([d7f1689](https://github.com/ysya/sshelter/commit/d7f16893d9bb80ee3af50e7397cfb6e133bdd03a))
* **keychain:** put the Keychain in the sidebar and remove the old Keys dialog ([284ba1f](https://github.com/ysya/sshelter/commit/284ba1f80e1e806cc8c859c3a1328727ccbeb11b))
* **keychain:** show a key's detail with its actions, and export a private key ([fc6da85](https://github.com/ysya/sshelter/commit/fc6da8537c7abcac731628f772402223b0aada7b))
* **keychain:** show keys only on this computer, when keys were made, and their passphrase ([9f79aab](https://github.com/ysya/sshelter/commit/9f79aabef4974f7b74c7b6216534d4e7d4bd65f9))
* **keys:** let a synced key live only in SSHelter on this computer ([5fd4b5b](https://github.com/ysya/sshelter/commit/5fd4b5bd3309760f291bf7d7e28cfe1e2d3a35e6))
* **keys:** list the hosts that use each key file and suggest Windows' OpenSSH for git ([0352633](https://github.com/ysya/sshelter/commit/0352633a6216c555d8ac8796e14b3b1923538c86))
* **relay:** self-host with Docker Compose on workerd behind Caddy ([8f313e2](https://github.com/ysya/sshelter/commit/8f313e263e4e82d3622eb68ff369c1c4eb61462d))
* **sidebar:** resize the sidebar by dragging its edge ([da31637](https://github.com/ysya/sshelter/commit/da316371cb9be55a9d7a0153119539b90303c289))
* **sync:** carry key slots through a sync code change ([f09e10c](https://github.com/ysya/sshelter/commit/f09e10cadb71c3d9cfe900a9cec1e4c603ebcebe))
* **sync:** key slot records, names and OpenSSH private key inspection ([a0cd33a](https://github.com/ysya/sshelter/commit/a0cd33a26fce1c1a399277eee77ab30e0f1ad45a))
* **sync:** keyslot and key records in the account chain ([bd6905f](https://github.com/ysya/sshelter/commit/bd6905fd0d74c8da38ee7b7278bdffdbb676748c))
* **sync:** land and maintain key slots on each computer ([dd00056](https://github.com/ysya/sshelter/commit/dd000561a45391048867c95664e1146c651d6bb6))
* **sync:** land synced keys in SSHelter's vault, with a file only when the vault can't be used ([96b5d0e](https://github.com/ysya/sshelter/commit/96b5d0ebcc783a1761bf0a3064a909fbe3b621ce))
* **sync:** let a key slot live only in SSHelter's vault ([f4a6ac2](https://github.com/ysya/sshelter/commit/f4a6ac2e5e9f78ebd064408c3d3691a65ddc2e56))
* **sync:** mark keys still kept as files and move them all into the vault ([333b32c](https://github.com/ysya/sshelter/commit/333b32cfc8f5320c81ff30e380569f0ae321c635))
* **sync:** offer a one-click relay deploy when no relay is set ([48fa754](https://github.com/ysya/sshelter/commit/48fa7548548c41193b8cc175e02d5ba7112e7f12))
* **sync:** offer keys in SSHelter's vault in the Sync key dialog ([9dd510a](https://github.com/ysya/sshelter/commit/9dd510a6984f5267273ca99ba9bf59a238425337))
* **sync:** owner-only key slot files, links and Windows ACLs ([a47af24](https://github.com/ysya/sshelter/commit/a47af2495cd7fca696720266f06ecf5ca3ee7575))
* **sync:** put picked and synced keys into the vault and keep the key a vault slot had ([f189761](https://github.com/ysya/sshelter/commit/f189761b31c59fc93db73383a0a4caf425f55945))
* **sync:** set up a key slot kept from a previous sync account in place ([b259ad1](https://github.com/ysya/sshelter/commit/b259ad1249abe9a08428f975bcc738244241bdcc))
* **sync:** set up key slots for synced hosts and rewrite their IdentityFile ([b770017](https://github.com/ysya/sshelter/commit/b770017c72249b294786d6fab7e41fd7041450b9))
* **sync:** Sync v2 with spaces, one sync code, a sync code change and approvals ([d51bfb5](https://github.com/ysya/sshelter/commit/d51bfb52408dc8283073ec751608d2028481e5c9))
* **sync:** sync, stop syncing, pick and replace keys in a slot ([07ba7b9](https://github.com/ysya/sshelter/commit/07ba7b9e22110abbbb61e596ef36f74033e4eb7a))
* **ui:** ask once per key whether it syncs when hosts start syncing ([1a28e49](https://github.com/ysya/sshelter/commit/1a28e490eb47a776aea86358d1a8c56e79522d99))
* **ui:** key slot data, helpers and dialog state ([19d1c4f](https://github.com/ysya/sshelter/commit/19d1c4f0613eb2ffe1e92909c9c83df223efe4ad))
* **ui:** manage key slots in Keys, Settings and the sidebar ([a1dd455](https://github.com/ysya/sshelter/commit/a1dd455f2d105d7556ac842f7742f9da0f94b678))
* **updater:** show the download's progress while an update installs ([54229c8](https://github.com/ysya/sshelter/commit/54229c8dbbcee33c66063bfce11491b5e740d15d))
* **vault:** add the encrypted key vault file ([c1e7669](https://github.com/ysya/sshelter/commit/c1e76690830f3ebd36a865cbf20779d8c4c8d02c))
* **vault:** export a private key to a file the user picks, optionally with a passphrase ([4ff1f68](https://github.com/ysya/sshelter/commit/4ff1f68e2cd5709d55b60740ba0bc91c52134d45))
* **vault:** keep a replaced key as a retired entry instead of dropping it ([e483825](https://github.com/ysya/sshelter/commit/e483825652286f000a27537e28d3876ebe1c6a57))
* **vault:** keep each new vault's key under a keychain account of its own ([ff4ed13](https://github.com/ysya/sshelter/commit/ff4ed13669c6452dba4476339dbcd2419d0d4e73))
* **vault:** parse, decrypt and sign with OpenSSH private keys ([77e87ab](https://github.com/ysya/sshelter/commit/77e87aba8556b7e77e18e3ce085a5982b5fa5f9c))


### Bug Fixes

* **agent:** accept only a login by the requested key and read every field to its end ([7b8883d](https://github.com/ysya/sshelter/commit/7b8883d2ac79f199657ec7caa539d062bbab433b))
* **agent:** arm the approval window so a second click cannot answer the next request ([d5b448b](https://github.com/ysya/sshelter/commit/d5b448bf5f0c991483fb5caa9685e3b860ee332a))
* **agent:** close an unused key channel, sweep leftovers and explain a Connect that can't use the key ([71a8978](https://github.com/ysya/sshelter/commit/71a8978e20b1f4a59556a4701553e25b67203b2f))
* **agent:** drop an opened key that no longer matches its slot and find a grant's key by slot id ([d5fbe6f](https://github.com/ysya/sshelter/commit/d5fbe6f666ad1bdcb34151fbb9c4bd9327fec01d))
* **agent:** fail closed on unreadable agent settings and tidy the endpoint ([dd6594d](https://github.com/ysya/sshelter/commit/dd6594debf6af40a93919e1962cdd6baab037669))
* **agent:** follow terminal tabs past login, keep scripts apart and keep only needed arguments ([0b0eb93](https://github.com/ysya/sshelter/commit/0b0eb9331c7b59a9a3dabd8af26b7bdc387c44e6))
* **agent:** keep a channel to tell a late ssh to connect again, and stay quiet when ssh never asks ([78bc468](https://github.com/ysya/sshelter/commit/78bc468a2e67d0e76638384ddcbd637afda64f41))
* **agent:** keep an answer that arrives as the prompt times out ([d9d8cdb](https://github.com/ysya/sshelter/commit/d9d8cdbe58c6dfeb23d92ac9eb9a57faa26432a6))
* **agent:** keep approval notifications in order and treat the approval window as its own ([dbae273](https://github.com/ysya/sshelter/commit/dbae2735011542cfc9a3f7f6077b2f56c405aec6))
* **agent:** keep spawned processes from holding the agent's sockets ([c4c701e](https://github.com/ysya/sshelter/commit/c4c701ed4ca44a6f870e41b8f0bd5e0d88db17ff))
* **agent:** let waiters outlast the first prompt and drop opened keys on time ([b787566](https://github.com/ysya/sshelter/commit/b787566efff69c5ecbe3b9aba1c8cf9789d2726d))
* **agent:** log an unsupported extension request's name and correct stale comments ([ccc164d](https://github.com/ysya/sshelter/commit/ccc164ddc0d9192711d87e7555e5e232a57811fb))
* **agent:** look up the IdentityAgent value only when agent/config is written ([123069d](https://github.com/ysya/sshelter/commit/123069db9baef948f07437a289e23b6140c7d1bb))
* **agent:** name replaced Linux executables and any-case .exe files correctly ([f3e3216](https://github.com/ysya/sshelter/commit/f3e3216793bd603fe998e31d9f2397b1d96f01a2))
* **agent:** refresh the agent config right after a host's IdentityFile changes ([5b5192d](https://github.com/ysya/sshelter/commit/5b5192d95013941a6b38a8481ec0ac23464554f6))
* **agent:** retry a failed first Include, refresh after every sync attempt and keep shared Include lines intact ([eb2ba28](https://github.com/ysya/sshelter/commit/eb2ba286c180b7bf60b3172679107f9a470195b1))
* **build:** embed the Windows app manifest in test binaries too ([c382df8](https://github.com/ysya/sshelter/commit/c382df84970b1f84fa137f9c8fb1bfe84a5603a5))
* **config:** quote an IdentityFile path that has spaces ([d3298fe](https://github.com/ysya/sshelter/commit/d3298fe2498a252f5c89596bb30773ff902b5ef1))
* **deploy:** re-read the config after a failed IdentityFile write so the retry works ([3d59c70](https://github.com/ysya/sshelter/commit/3d59c703ad820e11a2dad987dbbd94595330b407))
* **deploy:** read the host's keys before the attach line, and offer a retry ([6f127de](https://github.com/ysya/sshelter/commit/6f127de8e9379db37571333b1d27c3bc568c51d5))
* **keychain:** ask before New key moves a key file into SSHelter ([06377c2](https://github.com/ysya/sshelter/commit/06377c21f501e5d86aebb5845b928a2dd12addfe))
* **keychain:** check the vault's copy and the file again before a Move removes it ([387d216](https://github.com/ysya/sshelter/commit/387d216c6efabbbdba222bea02573044e4897903))
* **keychain:** don't delete a key file the loaded config can't vouch for ([6bb50d0](https://github.com/ysya/sshelter/commit/6bb50d051fbbb07ca7b0cde39f2082f7b23c9cd2))
* **keychain:** don't say a pasted key stays as a file ([a2a55dd](https://github.com/ysya/sshelter/commit/a2a55dd6c7c1ebceeca3c13ff3217378d48b9e36))
* **keychain:** keep listing this computer's keys in SSHelter without a sync account ([965a1e9](https://github.com/ysya/sshelter/commit/965a1e9fd531ad21aaee0caf2e4fd07044a91069))
* **keychain:** keep the passphrase optional in Export private key and show where a key file lives ([44b8d1a](https://github.com/ysya/sshelter/commit/44b8d1a40a16b2ee361af541168f55027efb1922))
* **keychain:** keep the user's pane when a slow New key finishes late ([e672037](https://github.com/ysya/sshelter/commit/e672037c3048032832a86f3893bcc97eff786022))
* **keychain:** keep unsaved host edits while the Keychain shows ([5643152](https://github.com/ysya/sshelter/commit/5643152c3de20f5d87ddda62988c8a281696d56b))
* **keychain:** let Delete copy remove an unused link without a sync account ([e60ffdf](https://github.com/ysya/sshelter/commit/e60ffdf5694274c071d8d545c0b98cd09d7ffc92))
* **keychain:** list keys only on this computer when the sync account's keys aren't loaded ([987a796](https://github.com/ysya/sshelter/commit/987a796f57a903fc7983f7aa61a24780c5a9e6ce))
* **keychain:** never delete a key file while something still points at it ([5d226c0](https://github.com/ysya/sshelter/commit/5d226c04f77bdce72d92a5fdd187c58245171270))
* **keychain:** point the missing key lint and the README at the Keychain ([d93df07](https://github.com/ysya/sshelter/commit/d93df07d0177bef9a0c2e72c55bd9d45bf1ec3d8))
* **keychain:** put a key lost from the vault back into its record ([d18a697](https://github.com/ysya/sshelter/commit/d18a697cbb468f8bd7b93d20ac6c73fc71e66f32))
* **keychain:** refresh slots after pointing a host at a key and share the overview mutation path ([4b8ea68](https://github.com/ysya/sshelter/commit/4b8ea686b6065dd2f6a3d3a31b103c585fa25449))
* **keychain:** refuse Move and Delete copy up front where the state can't be saved ([db9efbf](https://github.com/ysya/sshelter/commit/db9efbf1cc956100d50b9cbbfea04b53cbaa1aa5))
* **keychain:** report Move's failures even when the Keychain closes first ([2f8abf1](https://github.com/ysya/sshelter/commit/2f8abf1e1dacc898b497c8588a2663dfa8352d96))
* **keychain:** show a failed IdentityFile change with hidden characters revealed ([07642d9](https://github.com/ysya/sshelter/commit/07642d97a236497828c52800c8c2b538ada558de))
* **keychain:** show Hosts when a host is selected from anywhere ([a562ce8](https://github.com/ysya/sshelter/commit/a562ce8ca5f22babecce1fa722757d65f9bc1d9b))
* **keychain:** warn when dropped key files can't be heard ([fa65a01](https://github.com/ysya/sshelter/commit/fa65a014e7fbc72cb119dc268f88bf80f51a906d))
* **keychain:** wrap the Move confirm's path, title a failed Delete key, ignore a late Move press ([f49709c](https://github.com/ysya/sshelter/commit/f49709cfe961fa0265a0c3cd0fb9edbbfcb66034))
* **keys:** let Fix finish wiring the agent and tidy the vault rows ([4d7df86](https://github.com/ysya/sshelter/commit/4d7df86d146e6fdd4865be8a6c6698991ebe6d54))
* **mcp:** bound the wait for mcp-spawn.lock and start without it if it can't be taken ([e867af9](https://github.com/ysya/sshelter/commit/e867af9712b8e86eed9493b1d2c80e78fe293151))
* **mcp:** bring back the window when an adapter-started SSHelter restarts ([e70f412](https://github.com/ysya/sshelter/commit/e70f412f5146441b37abc204eb6fe34a9360ab91))
* **mcp:** keep the bridge accepting after a failed accept ([edf5350](https://github.com/ysya/sshelter/commit/edf5350f5dd281d7f7c8ffa0ebffe2caccad39f9))
* **mcp:** keep the MCP host running when the AI tool ends its session ([740e2b8](https://github.com/ysya/sshelter/commit/740e2b8dbe311cde576dcdd6477d52387c0d4ed5))
* **mcp:** let a debug build run next to the installed SSHelter ([7bd24d3](https://github.com/ysya/sshelter/commit/7bd24d3e029fe41c97eb24f049d1b7ec8231cf1c))
* **mcp:** remove the MCP runtime file when its SSHelter exits ([415be8b](https://github.com/ysya/sshelter/commit/415be8bcd40bf9f5091593b1ba18791a2cf129fd))
* **mcp:** run one SSHelter at a time ([37ba1a9](https://github.com/ysya/sshelter/commit/37ba1a94e5c9aef93d9e87d0db67ed3fb393b573))
* **mcp:** say what to do when SSHelter doesn't answer ([a449e5f](https://github.com/ysya/sshelter/commit/a449e5fb266feadf5db13183aa1ca1f7f7d0e566))
* **mcp:** start SSHelter once when parallel calls find no bridge ([07155e0](https://github.com/ysya/sshelter/commit/07155e09ec6b2b83b6966c93b307b68c2702829d))
* **mcp:** start the MCP host in the background ([0e2ccb7](https://github.com/ysya/sshelter/commit/0e2ccb7d714be278bdec03a6d5cbe0b8f9e7d5d8))
* **sync:** accept only a bare public key in keyslot records ([18d7f77](https://github.com/ysya/sshelter/commit/18d7f77bad7193ad525a2ddef3da7c9dc0c5601b))
* **sync:** aligned token buffer and no leftover temp link on Windows ([5f827f4](https://github.com/ysya/sshelter/commit/5f827f46238b827ed96b4e137c6323d4d96b5d90))
* **sync:** commit a key's move into the vault before its file goes, and keep vault slots consistent ([8267af2](https://github.com/ysya/sshelter/commit/8267af282eadb1147db8fd9dbc2c35adec3145f3))
* **sync:** delete only SSHelter's own copy and report key changes only to the uploader ([354ae61](https://github.com/ysya/sshelter/commit/354ae6145521fb6e24291d8a8b82c4349cc32e94))
* **sync:** forget a slot's remembered passphrase when its key leaves the vault ([575fe29](https://github.com/ysya/sshelter/commit/575fe29a43287660aad690fbbab0581a31bb5c1d))
* **sync:** forget a vault slot's remembered passphrase when its key changes ([ae35ab8](https://github.com/ysya/sshelter/commit/ae35ab8fa2dd4ff2bcd3f88160f317a5f700796d))
* **sync:** keep a key slot while any host on this computer uses it ([a24df3b](https://github.com/ysya/sshelter/commit/a24df3b3a9e528524d69618ac404c4516fd2e6bb))
* **sync:** keep a key the agent cannot read out of the vault ([0ae3fcc](https://github.com/ysya/sshelter/commit/0ae3fcc671b4486c3b11ef7857a66db8b3e86b81))
* **sync:** keep key slot consent for the computer that changes the sync code ([fc7c6fe](https://github.com/ysya/sshelter/commit/fc7c6fedfc480db25543cee65c60a08f7e711f08))
* **sync:** keep key slot records while unused and re-upload only keys synced here ([42996f7](https://github.com/ysya/sshelter/commit/42996f721c21286e6846b56c08885224106b61e1))
* **sync:** keep key sync consent when rejoining the same account ([3879a3c](https://github.com/ysya/sshelter/commit/3879a3c2c4e477263ddff430ef6bf736e60ed915))
* **sync:** keep keys the agent cannot sign with out of the vault ([f06711f](https://github.com/ysya/sshelter/commit/f06711f9400add7cf2375a7d9e20385891426ad8))
* **sync:** keep this computer's keys maintained without a sync account ([d1e644d](https://github.com/ysya/sshelter/commit/d1e644dea46edba6eaaca184b848e5705af303ec))
* **sync:** keep Windows key slot ACLs off the user's own key files ([1106ed3](https://github.com/ysya/sshelter/commit/1106ed3564b44d0dd75b9bef6672cb786972880c))
* **sync:** land a reused vault key through the vault and report an unreadable vault as itself ([56e7c0d](https://github.com/ysya/sshelter/commit/56e7c0dca04b803325c266787d0e6cc7fdff0c56))
* **sync:** leave keys SSHelter's agent can't hold out of File for now and Move ([ccbfcc8](https://github.com/ysya/sshelter/commit/ccbfcc85a739100e4428662849191bfeb50189c3))
* **sync:** look after this computer's own keys at the end of every sync attempt ([f1b6a61](https://github.com/ysya/sshelter/commit/f1b6a618eaba49b88748242b8878a481dfd0cf8e))
* **sync:** move only the key slots a setup creates into SSHelter ([8094a9d](https://github.com/ysya/sshelter/commit/8094a9dfce8d4cd5483affd56a85f774b5363ee6))
* **sync:** never carry a key's sync consent into another account ([6d39586](https://github.com/ysya/sshelter/commit/6d39586c0f9882f4e33ebf3beba4acd23d5faaea))
* **sync:** never land a key slot whose file name another slot holds ([e58387d](https://github.com/ysya/sshelter/commit/e58387d7921f29d479b2fc00d3cb7406e5263367))
* **sync:** never relink a parked key slot over another file ([79adfe3](https://github.com/ysya/sshelter/commit/79adfe3d106d3340357c2e5cf0ae2cda9df7b441))
* **sync:** never republish a key only on this computer to the account ([5cf2263](https://github.com/ysya/sshelter/commit/5cf226399d684d89de5e79db5b936d6348e068ab))
* **sync:** pick up keyslot records an older build kept sealed ([338ebf7](https://github.com/ysya/sshelter/commit/338ebf7de1acddf6c0e1db67df47b942d3a38af8))
* **sync:** refuse key slot changes during a sync code change ([f4eeaa2](https://github.com/ysya/sshelter/commit/f4eeaa21d7debedd7326c08a54718022a4413c10))
* **sync:** rewrite a linked slot's .pub from its own key ([ff23371](https://github.com/ysya/sshelter/commit/ff2337132ab28916dff2a47a3c9436c0b6d21f78))
* **sync:** set up key slots only in spaces that finished their first sync ([f5461e1](https://github.com/ysya/sshelter/commit/f5461e1bac1a81d1ed8b1cd15e0cca73ce373946))
* **sync:** update the agent config when a picked key falls back to a link ([096815c](https://github.com/ysya/sshelter/commit/096815c71923c5612abed8e33b1a1e903d3e5946))
* **sync:** upload a kept copy only with consent and land a reused slot first ([d0e7b49](https://github.com/ysya/sshelter/commit/d0e7b49a2cf596d3330a4533b7e2d46327166a68))
* **tray:** unminimize the window from Open SSHelter ([f402a55](https://github.com/ysya/sshelter/commit/f402a55fa763bf3077212368c7ff47eba7f3a255))
* **ui:** ask about keys after an update only once the config has loaded ([b8a8924](https://github.com/ysya/sshelter/commit/b8a89240072b77b2af3c9759f2305c591270cc13))
* **ui:** confirm before a key syncs and keep slot actions honest ([81dcb47](https://github.com/ysya/sshelter/commit/81dcb47b42fe5d2e1472f5c8c0bd72aa6bf8e681))
* **ui:** don't submit when Enter commits an IME composition ([aa41d31](https://github.com/ysya/sshelter/commit/aa41d31071371cbd2b4b23f59399646ed1c1a201))
* **ui:** keep dialog content within the dialog's width ([6f8afe4](https://github.com/ysya/sshelter/commit/6f8afe40add423928017699a5bafc0c66564c211))
* **ui:** keep the key dialog busy while it re-reads and ask after enabling an IdentityFile ([1d5e61a](https://github.com/ysya/sshelter/commit/1d5e61afb6ec96b594f9b8260e73ac31096eb6f2))
* **ui:** offer a key pick for a key slot in error ([3688c4a](https://github.com/ysya/sshelter/commit/3688c4ab9e41fe87afa4bdff9f948aeeb08cf778))
* **ui:** reveal hidden characters in key slot errors ([8789150](https://github.com/ysya/sshelter/commit/8789150c30223d2d735de29d7cfbf9c5a101ecd6))
* **ui:** scroll the key slot lists and show the file each slot uses ([67de55e](https://github.com/ysya/sshelter/commit/67de55e735b5731ea61c38490ff916e9087d6458))
* **ui:** write ~/.ssh/... only for keys in this user's own .ssh ([f0d527a](https://github.com/ysya/sshelter/commit/f0d527a3b8944dc04c2452d537227af8d05aead0))
* **vault:** keep newer vault files in place and set aside a vault whose key is gone ([9211912](https://github.com/ysya/sshelter/commit/921191235638f6681eb863096c15eecde1b16fb8))
* **vault:** write an exported key owner-only from its first byte, without a temp file ([723c244](https://github.com/ysya/sshelter/commit/723c24445273382a5dbf37e3cde2a9f91a8af577))
* write Windows key paths as ~/.ssh and explain missing key slots ([1b62bc9](https://github.com/ysya/sshelter/commit/1b62bc9909480623c3de43d01ff5a6e72a264c80))


### Performance Improvements

* **keys:** list keys off the main thread ([0eecc41](https://github.com/ysya/sshelter/commit/0eecc410ee14c98fb856041a16b627cebd710195))

## [0.16.0](https://github.com/ysya/sshelter/compare/v0.15.1...v0.16.0) (2026-10-01)


### Features

* **ci:** beta channel script that validates beta versions and maintains the updater-beta manifest ([88a9f48](https://github.com/ysya/sshelter/commit/88a9f480f6c617d45900971a81af6ed84b8ce021))
* **relay:** zero-knowledge sync relay on Cloudflare Durable Objects ([c346f83](https://github.com/ysya/sshelter/commit/c346f83fe11a9efdd5b90b5af8d2079b26cd54b4))
* **sync:** background engine with transactional apply, save-time planning and lifecycle mutex ([44a17ff](https://github.com/ysya/sshelter/commit/44a17ff6254e70ceb705a053143f471e46ce95b0))
* **sync:** detect local host block changes with per-block change times ([26390a4](https://github.com/ysya/sshelter/commit/26390a43bfae3a241d670b97fa7c5ac86f2b9315))
* **sync:** expose first_sync_pending; the wizard moves only visible hosts and refetches after failed writes ([c6cf129](https://github.com/ysya/sshelter/commit/c6cf1299d7aba1b277121ea955b9f05fe91f64fd))
* **sync:** frontend hooks, event wiring and migration helpers ([e631b7b](https://github.com/ysya/sshelter/commit/e631b7b638c9748ca0382070987833cd4a4316d7))
* **sync:** let release builds ship without a built-in relay and ask for one in Settings → Sync ([29cd804](https://github.com/ysya/sshelter/commit/29cd804bcfec3befd64dcf7e2b52e683c5f3a54c))
* **sync:** managed hosts file block operations ([b97870f](https://github.com/ysya/sshelter/commit/b97870f2a1e02ea5cc1e58cafbf38fdcde1cf37c))
* **sync:** migrate hosts into the synced file, detect and resolve shadowed aliases by file ([432fe05](https://github.com/ysya/sshelter/commit/432fe055c191b54afb7a25b754d3422798bbd2c0))
* **sync:** migration wizard with file-addressed shadowed alias handling ([f3f3da8](https://github.com/ysya/sshelter/commit/f3f3da8ceea13802bfe9052ebd6570f3e5d206c0))
* **sync:** mnemonic, key derivation and record encryption ([97aa368](https://github.com/ysya/sshelter/commit/97aa368984435d82c11e8351f3b01727ef585b7f))
* **sync:** persist local sync state and keep the mnemonic in the keychain ([e002dc0](https://github.com/ysya/sshelter/commit/e002dc0af9b23ba869ce9241285f7579c82a13c7))
* **sync:** plan/pull-merge/push round with sealed retention and batched uploads ([5f6354b](https://github.com/ysya/sshelter/commit/5f6354b345e9a98fb9e360ad28f30e1291f960a8))
* **sync:** record model, wire envelope and last-writer-wins merge ([6e51d67](https://github.com/ysya/sshelter/commit/6e51d672b3e2fc9e748c000209e9a6000e926d4b))
* **sync:** relay HTTP client with conflict-aware push ([2783365](https://github.com/ysya/sshelter/commit/27833659b9c5856b33c365615ee1a6242d811e79))
* **sync:** settings pane to create, join, inspect and leave a sync chain ([df511a4](https://github.com/ysya/sshelter/commit/df511a48d72fad6a11e3770703fe1ca90a19e7f0))
* **updater:** backend commands to check and install from the Beta update channel ([a653bf7](https://github.com/ysya/sshelter/commit/a653bf777fb2743efff119aeaec355a5cd221624))
* **updater:** Stable/Beta update channel setting and mark sync as beta ([906dff0](https://github.com/ysya/sshelter/commit/906dff0aa821405b2c10ce19b45bb6e34c652fc6))


### Bug Fixes

* **ci:** enforce the exact beta version format and harden the beta channel script ([6a82bea](https://github.com/ysya/sshelter/commit/6a82bea316f07e33c68f23408db04cbbe7d54022))
* **ci:** enforce Windows MSI version limits on betas and clarify republishing ([62fbb16](https://github.com/ysya/sshelter/commit/62fbb16636353a09aad57daf87eedebe63704390))
* **ci:** make build-platform refuse beta tags before building anything ([a6de115](https://github.com/ysya/sshelter/commit/a6de1150a094f63e331256f2c2c7500460c6d2cc))
* **ci:** refresh the Beta manifest when a version is rebuilt and validate it before promoting ([e6f622e](https://github.com/ysya/sshelter/commit/e6f622e806fe50d52c85feaf4e024588b4aa92fa))
* **ci:** refuse existing beta tags, keep betas prerelease and reject leading zeros ([ff9f82b](https://github.com/ysya/sshelter/commit/ff9f82b291cd2d4e80950dc54eb11cf89cd2ee94))
* **ci:** refuse to publish a beta when the tag check cannot reach origin ([e9e1aca](https://github.com/ysya/sshelter/commit/e9e1acad85b50b807e467a01236503a7fdc35a63))
* **sync:** drop a stale ts-expect-error and cover lone ? wildcards ([4c4eeee](https://github.com/ysya/sshelter/commit/4c4eeeebd3a3cb317f2fd8165e4e64969ad0d46e))
* **sync:** fall back to the local relay when the injected relay URL is empty ([5d3911e](https://github.com/ysya/sshelter/commit/5d3911ec2ff43438814986ff629ce59bb02bb927))
* **sync:** keep synced names out of the wizard, wait for the first sync and show sync errors ([35de116](https://github.com/ysya/sshelter/commit/35de11607bcf096977252c22983e17bc3e80a626))
* **sync:** keep the leave dialog open until the request settles ([ed67453](https://github.com/ysya/sshelter/commit/ed67453590e16c62a5b133d241905c1743f733b1))
* **sync:** keep unpushed edits when restoring the synced file and repair a rolled-back relay ([e6a1945](https://github.com/ysya/sshelter/commit/e6a1945c1d7983ae0709d824a7ad5586ff7b3b17))
* **sync:** list failed host moves and distinguish loading from empty in the migration wizard ([f95f451](https://github.com/ysya/sshelter/commit/f95f4518c94b7b5e44a57eb863f60e4f976ad8c5))
* **sync:** one engine per OS user, bounded conflict retries and safer sync rounds ([1baf929](https://github.com/ysya/sshelter/commit/1baf9297e4c80aa88a4b7add5b10bfeb1fd7e1fe))
* **sync:** quiet startup before the config loads, restore a vanished synced file from the chain and harden state and keychain errors ([94e788d](https://github.com/ysya/sshelter/commit/94e788ddb7606fbd0fbb4385ee5f3bf240b5b6a2))
* **sync:** refuse moves that share any synced name or run without the engine, reload after a failed shadow fix ([cdbf943](https://github.com/ysya/sshelter/commit/cdbf943589ace707229aa997ab0f0282adebb83c))
* **sync:** refuse moves that would duplicate a synced alias and reload after a failed move ([657e161](https://github.com/ysya/sshelter/commit/657e1615b0a5e6bb31772a820ba123160960cf0b))
* **sync:** retry a held engine lock briefly and wake the engine after every config load ([3e4783f](https://github.com/ysya/sshelter/commit/3e4783f26407d1fc48e395ec665ee53ad8d49039))
* **sync:** stop a host migration batch at the first failed write and reload from disk ([c263572](https://github.com/ysya/sshelter/commit/c2635728314cdc9cc1009f99f50cbe84aab25270))
* **sync:** surface engine doc reloads, keep unreadable state files and emit outside the lifecycle lock ([0daa74a](https://github.com/ysya/sshelter/commit/0daa74a6ac97932349d0b1d50f52996e01bad9bb))
* **sync:** treat unreadable metadata as an error, never as a missing synced file or state ([305f0a9](https://github.com/ysya/sshelter/commit/305f0a917e153fa33416df2b1fb9fcdb65b02e65))
* **sync:** wake the engine only when a config load changes the synced file, report baseline conflicts, clarify secondary-name refusals ([2661b13](https://github.com/ysya/sshelter/commit/2661b13b26121f5ef0fe983726929f1a5d448db0))
* **updater:** ignore stale channel results and retire the update prompt on any channel change ([ca66e3a](https://github.com/ysya/sshelter/commit/ca66e3ad48291be2b694c10fd5fbc5444cbfe71f))

## [0.15.1](https://github.com/ysya/sshelter/compare/v0.15.0...v0.15.1) (2026-09-01)


### Bug Fixes

* **clipboard:** use native clipboard on Windows ([1052b21](https://github.com/ysya/sshelter/commit/1052b21f4e559366c68dfc5a6e1076f72f282155))

## [0.15.0](https://github.com/ysya/sshelter/compare/v0.14.0...v0.15.0) (2026-08-31)


### Features

* **connect:** auto-fill saved passwords when connecting ([8cc2542](https://github.com/ysya/sshelter/commit/8cc25426b248e8a2408808894ad0508e276835bc))


### Bug Fixes

* **deploy:** make in-app key deploy work on Windows ([2b1319e](https://github.com/ysya/sshelter/commit/2b1319e16c94354be69bf3b3ea9837ac77762df5))

## [0.14.0](https://github.com/ysya/sshelter/compare/v0.13.0...v0.14.0) (2026-08-24)


### Features

* **hosts:** add SSH name copy and fix Windows commands ([65f65c9](https://github.com/ysya/sshelter/commit/65f65c92eae3b807b8f8a5d23c619ca674ecbb5f))

## [0.13.0](https://github.com/ysya/sshelter/compare/v0.12.0...v0.13.0) (2026-08-14)


### Features

* **keys:** hint how to enable the Windows ssh-agent service ([122b0a3](https://github.com/ysya/sshelter/commit/122b0a3c1f5d762a4844abd5fb48120ac37220a1))
* **mcp:** add UI-controlled SSH access ([963467e](https://github.com/ysya/sshelter/commit/963467e5b5d26b8d9d78ea6e56a27e1ca14f7836))

## [0.12.0](https://github.com/ysya/sshelter/compare/v0.11.0...v0.12.0) (2026-08-14)


### Features

* **windows:** build Windows installers and launch wt/cmd terminals ([455fa6b](https://github.com/ysya/sshelter/commit/455fa6ba61b6a38479a0e4e8f5e17ff20f8c3d37))

## [0.11.0](https://github.com/ysya/sshelter/compare/v0.10.0...v0.11.0) (2026-08-13)


### Features

* **config:** new-config-file dialog with live include preview ([9c3ac32](https://github.com/ysya/sshelter/commit/9c3ac32b1b58e87f70f2b3ce9dffae42004d25c0))
* **config:** plan and create included config files ([f1fe215](https://github.com/ysya/sshelter/commit/f1fe215a8d16d4b49e8fcef3a3b55cbad4a17561))
* **host-list:** create a new config file from every file picker ([b3668d0](https://github.com/ysya/sshelter/commit/b3668d0df7a17fd8e2ad62014ae1d64a927dba4d))

## [0.10.0](https://github.com/ysya/sshelter/compare/v0.9.0...v0.10.0) (2026-08-13)


### Features

* **host-list:** drag a host onto another file group to move it ([453593f](https://github.com/ysya/sshelter/commit/453593f268643e1de4219e58fc1f2082defa4676))
* **host-list:** hover actions menu with move and remove ([efb67e4](https://github.com/ysya/sshelter/commit/efb67e48fd620d864669d66be3513274ad0968ee))
* **host-list:** multi-select with batch move, tag and remove ([c8e375a](https://github.com/ysya/sshelter/commit/c8e375a7cddb8153bda3d9c33e67d22c0109d01f))

## [0.9.0](https://github.com/ysya/sshelter/compare/v0.8.0...v0.9.0) (2026-08-13)


### Features

* **host-list:** add #tag [@user](https://github.com/user) search prefixes ([9e26e08](https://github.com/ysya/sshelter/commit/9e26e083f09243a83e539a116d22acfd10dfbfa2))
* **host-list:** group hosts by file or by tag ([4bc643b](https://github.com/ysya/sshelter/commit/4bc643b3ba6f9344e6e509826fc88688e574632c))
* **host-list:** show tag chips on host rows ([9c7b905](https://github.com/ysya/sshelter/commit/9c7b905a3fba7867087fc77e8fae0f1262d76e36))
* **palette:** surface recent connections first ([daae0d2](https://github.com/ysya/sshelter/commit/daae0d297a3dd5f8a068ff16c51ab860e80fb87b))


### Bug Fixes

* **host-editor:** demote deploy button to the menu once a key is configured ([29df22f](https://github.com/ysya/sshelter/commit/29df22f7e95b7fad509538bd08144e94cafff8d6))

## [0.8.0](https://github.com/ysya/sshelter/compare/v0.7.0...v0.8.0) (2026-08-13)


### Features

* **deploy:** add editor, palette, hygiene and keys-dialog entry points ([b73501e](https://github.com/ysya/sshelter/commit/b73501e0b218cb5bccd870ae8f392f965b7c4677))
* **deploy:** add identity-file write-back decision helpers ([c690d74](https://github.com/ysya/sshelter/commit/c690d7450e452535e2443e5dad2e3792d9471e2a))
* **deploy:** write IdentityFile back after a successful deploy ([7c3169f](https://github.com/ysya/sshelter/commit/7c3169fb7c3248e605ceb9b922771c88f50fdca1))
* **host-editor:** pick IdentityFile from detected keys or a file dialog ([e939e1a](https://github.com/ysya/sshelter/commit/e939e1a24b86126678d5edff69ae141abd291d14))


### Bug Fixes

* **ui:** disable autocorrect and autocapitalize on all text inputs ([d806a66](https://github.com/ysya/sshelter/commit/d806a66ba7afcd59a1705df1f995ad493ec8f502))

## [0.7.0](https://github.com/ysya/sshelter/compare/v0.6.1...v0.7.0) (2026-08-13)


### Features

* **askpass:** add SSH_ASKPASS helper mode with prompt whitelist ([e413437](https://github.com/ysya/sshelter/commit/e413437095fcb199c2d5517e229fc242d0cfd2e0))
* **askpass:** dispatch to helper mode before Tauri init ([be79556](https://github.com/ysya/sshelter/commit/be79556b1ff7c295766123fa477fdcd8c7c90d54))
* **deploy:** add deploy/precheck/secrets Tauri commands ([1b1b449](https://github.com/ysya/sshelter/commit/1b1b4491f09c6f26de1491b92bc817bf8d72052e))
* **deploy:** add host key precheck against known_hosts ([dbd93c0](https://github.com/ysya/sshelter/commit/dbd93c0b4bf7633281678dad90d1b3e4e6a21cb6))
* **deploy:** add in-app key deployment dialog ([b3c3780](https://github.com/ysya/sshelter/commit/b3c3780a63944832f5a00e69fb2ffad32b70520d))
* **deploy:** add pure argv builder, remote script and outcome classifier ([a343133](https://github.com/ysya/sshelter/commit/a343133f5e57d6b416d123c681744a44db4de36e))
* **deploy:** warn about old OpenSSH and password-blocking config ([94c8b19](https://github.com/ysya/sshelter/commit/94c8b1981de7d575fa2c7550b4a0214d71203744))
* **host-editor:** manage the host password stored in the OS keychain ([79f70fd](https://github.com/ysya/sshelter/commit/79f70fdd0c74817179c92b84506ce70594892542))
* **host-list:** right-click a host to deploy a key ([777fe89](https://github.com/ysya/sshelter/commit/777fe89d99b2968278b6bc6e232eb86f6683ea53))
* **queries:** add deploy and host-password hooks ([d6a89a9](https://github.com/ysya/sshelter/commit/d6a89a9ba8ffab921ef7715d51c84814134e1eed))
* **secrets:** add OS keychain wrapper for per-host passwords ([be59a76](https://github.com/ysya/sshelter/commit/be59a76211a2c456fed964edde808c784f8ef3a5))


### Bug Fixes

* **askpass:** anchor prompt whitelist to real OpenSSH client behavior ([325b34c](https://github.com/ysya/sshelter/commit/325b34ce49810ad6cf3ba54e4dd4f749c141fd6e))
* **askpass:** correct module doc and use lossy argv decoding ([e1a3c68](https://github.com/ysya/sshelter/commit/e1a3c683729249182c093aed44bacf75a22dd4cf))
* **deploy-ui:** trust the host key by confirmed fingerprint, not key line ([d0ee77d](https://github.com/ysya/sshelter/commit/d0ee77d28aac33e279a2268652378752c03b430b))
* **deploy:** close config/timeout/keychain gaps in the deploy commands ([a075bbd](https://github.com/ysya/sshelter/commit/a075bbdb81d51166d3d2ea41f2159d9fce45aeae))
* **deploy:** guard authorized_keys corruption and misclassified auth failures ([aac2fc9](https://github.com/ysya/sshelter/commit/aac2fc9e228c37a6a3b893a401ecad192c146c33))
* **deploy:** parse known_hosts markers so CA-trusted and revoked hosts are handled correctly ([646ac75](https://github.com/ysya/sshelter/commit/646ac75f278335787b78a620de8e4a11d9dbe1e1))
* **secrets:** trust any keyring error as unavailable, guard test cleanup ([5f1e44a](https://github.com/ysya/sshelter/commit/5f1e44a4f264614ed4725c3b0f267871d678e66a))

## [0.6.1](https://github.com/ysya/sshelter/compare/v0.6.0...v0.6.1) (2026-06-19)


### Bug Fixes

* **host-list:** show the Defaults group at the top of each file section ([022087a](https://github.com/ysya/sshelter/commit/022087a90dd58c55e1c72338e5229e472d08f9c6))

## [0.6.0](https://github.com/ysya/sshelter/compare/v0.5.1...v0.6.0) (2026-06-19)


### Features

* **host-list:** add target-file resolver for right-click add-host ([3934ab1](https://github.com/ysya/sshelter/commit/3934ab1d50f41658c75bcedc37aa41269bb45dbd))
* **host-list:** right-click file headers to add a host, view, or rename ([b809e95](https://github.com/ysya/sshelter/commit/b809e95ddf5bcc81eee4d984ee46ea920777e036))
* **host-list:** seed AddHostDialog target from right-click file ([c71a7a7](https://github.com/ysya/sshelter/commit/c71a7a75c2fa41bf2841fd73a6e479577af2e2c1))
* **host-list:** track right-click add-host target file in ui store ([33288f2](https://github.com/ysya/sshelter/commit/33288f2aa0821a2d8311a2cab5a960a33cbe6788))
* **ui:** add context-menu primitive ([80155e0](https://github.com/ysya/sshelter/commit/80155e03ac7d3061c285b4fe6144c6d4ae8684fc))

## [0.5.1](https://github.com/ysya/sshelter/compare/v0.5.0...v0.5.1) (2026-06-12)


### Bug Fixes

* **ui:** keep toasts clickable above modal dialogs ([cf668fd](https://github.com/ysya/sshelter/commit/cf668fddba13bbb787b61384c8fb98a2fe29ff88))
* **updater:** keep checking for updates while the app runs ([2b0a1ab](https://github.com/ysya/sshelter/commit/2b0a1aba5990e1d7a8ba7fdc1ea8ea598e8b1fa9))

## [0.5.0](https://github.com/ysya/sshelter/compare/v0.4.0...v0.5.0) (2026-06-11)


### Features

* **app:** launch-at-login, global hotkey, settings export/import, ⌘F/⌘N ([636144b](https://github.com/ysya/sshelter/commit/636144b7dfec191bc515ff7b7db0776235ce86b0))
* **hosts:** move/duplicate host across files, per-host terminal override, raw file viewer ([4eb8e52](https://github.com/ysya/sshelter/commit/4eb8e52751db1dc8a29733f1497b4cd4b162801c))
* **keys:** SSH key management — list/fingerprints/agent status, generate ed25519, copy pubkey, ssh-copy-id deploy via terminal ([c54f1ab](https://github.com/ysya/sshelter/commit/c54f1ab1038a8154dca0ab2c3cc3f869d09b32b7))
* **known-hosts:** known_hosts viewer — search + safe entry removal (lossless, backed up) ([0130f51](https://github.com/ysya/sshelter/commit/0130f510fbddd2f9bd375a914cef7191fbc63a7d))
* **ui:** drag-to-reorder hosts within a config file ([2226433](https://github.com/ysya/sshelter/commit/2226433cf3cba13036fe7df03a76c23f5799820e))
* **ui:** user-adjustable text size (Settings &gt; Appearance, scales the rem-based UI) ([09b9d2a](https://github.com/ysya/sshelter/commit/09b9d2ab0e316399af4826c18f0f4a5d307b3d86))


### Bug Fixes

* **config:** address option toggles by occurrence index (same-keyword pairs hit the wrong line) ([bbbf10f](https://github.com/ysya/sshelter/commit/bbbf10f96c5506418c59881504ffd274976cf41d))
* **ui:** overlay buttons centered with translate (-translate-y-1/2) lost ([4eb8e52](https://github.com/ysya/sshelter/commit/4eb8e52751db1dc8a29733f1497b4cd4b162801c))

## [0.4.0](https://github.com/ysya/sshelter/compare/v0.3.0...v0.4.0) (2026-06-11)


### ⚠ BREAKING CHANGES

* **config:** existing next-to-file .bak snapshots are moved into the new backups root on first load; restore only accepts backups inside the new location.

### Features

* **config:** rename host — lossless Host-line pattern editing from the editor header ([6ea2e1a](https://github.com/ysya/sshelter/commit/6ea2e1a979cb0e482cbde85ea99b635fff0ad4da))
* **ui:** sidebar v2 — file scope filter, compact rows, sticky headers, wildcard defaults footer, persisted nav state ([0e7cb97](https://github.com/ysya/sshelter/commit/0e7cb97b00d0430dae6a19ea1ba68812d0f05414))
* **ui:** user-defined display aliases for config file groups (double-click to rename) ([b450afc](https://github.com/ysya/sshelter/commit/b450afce1a5c1651781a3a438ee20ce56e807995))


### Bug Fixes

* **config:** relocate backups out of ssh-visible dirs (glob Includes were loading .bak files) ([c7c4707](https://github.com/ysya/sshelter/commit/c7c47078a24c427bfb12bfe5f54b70c4bb63e706))
* **ui:** label colliding config files by their distinctive ancestor (orbstack, not ssh/config) ([0c236ac](https://github.com/ysya/sshelter/commit/0c236ac891a94332044ad5f3ef4a97d551c76ff9))

## [0.3.0](https://github.com/ysya/sshelter/compare/v0.2.0...v0.3.0) (2026-06-10)


### Features

* **app:** auto-update via tauri-plugin-updater ([10f096b](https://github.com/ysya/sshelter/commit/10f096ba5f193d2ade835cb6ab8dbb51fa9851a9))

## 0.2.0 (2026-06-10)


### Features

* add app_platform smoke command and wire error/fsutil modules ([55b35a2](https://github.com/ysya/sshelter/commit/55b35a2e26d0ca40f42a5fbf9e90129694ee95de))
* add AppError unified command error type ([af03563](https://github.com/ysya/sshelter/commit/af03563bcd9af9692e4b9bd53fb69b2f44114d6d))
* **config:** app state + config_* Tauri commands + safe write path ([5647071](https://github.com/ysya/sshelter/commit/5647071468752d17298580d52da12c9417b16fb7))
* **config:** backup history listing + safe restore ([26cc242](https://github.com/ysya/sshelter/commit/26cc242d88a8a17153850232e0d0a315a4fec071))
* **config:** CST edit operations (set/add/remove/toggle/reorder/group) ([a4a5099](https://github.com/ysya/sshelter/commit/a4a5099a5d46b8fd35d42a0a0eb387201e6f4edf))
* **config:** CST model + quote-aware single-line lexer ([fa67dec](https://github.com/ysya/sshelter/commit/fa67dec2bce93f63bcfc9a147200f94fbe13859c))
* **config:** intelligence module — effective-config (ssh -G), linter, ProxyJump chain, key-hygiene ([3a70ba8](https://github.com/ysya/sshelter/commit/3a70ba8ba946ffeb9e12e0fa81ca5bd13dfe1c96))
* **config:** lossless parser + serializer with golden round-trip corpus ([de4b150](https://github.com/ysya/sshelter/commit/de4b150f51b52cffc1ba219f602bbbbc63e96346))
* **config:** multi-file Include loading + host DTOs ([2d196b6](https://github.com/ysya/sshelter/commit/2d196b6fe68e9878f538ee936391343c31673948))
* **connect:** terminal launcher with per-emulator argv + alias validation ([05f10f1](https://github.com/ysya/sshelter/commit/05f10f110b949328d8f1ab58a894b7f1e2e0d3b2))
* **core:** settings backend — lint rule ids, backup retention, tray toggle, close-to-tray, iTerm2 new-tab ([c92ed27](https://github.com/ysya/sshelter/commit/c92ed27ac5a073230f84580afe1ae70990cb9f09))
* **discover:** known_hosts + Tailscale host discovery ([c861d81](https://github.com/ysya/sshelter/commit/c861d81a664dd6994c5c3d9092719c138aad6139))
* **error:** add AppError::Parse variant ([7136393](https://github.com/ysya/sshelter/commit/7136393551de59c48b60a3bb32603fc5c1ec638e))
* establish ts-rs Rust-&gt;TS type bridge (Fingerprint) ([9d36ce3](https://github.com/ysya/sshelter/commit/9d36ce3be6e98f17a097fb76ea3ee5f57c99b479))
* **fsutil:** fsync parent dir after rename; document symlink/perms semantics ([43db87b](https://github.com/ysya/sshelter/commit/43db87b8ddb0129b7a0885ee145a9509b3740c0a))
* **fsutil:** safe file IO — atomic write, perms, backup, drift detection ([95d273b](https://github.com/ysya/sshelter/commit/95d273bf432ebcd994275fdc93dbc4820df246ec))
* **tray:** menubar quick-connect menu rebuilt on config load ([bae8a6e](https://github.com/ysya/sshelter/commit/bae8a6ea38430ff87e51378c7afc7b0e4754e8fb))
* **ui:** center the host editor at a comfortable column width (System Settings style) ([e04ddd4](https://github.com/ysya/sshelter/commit/e04ddd450b3b41d8e73bbbabca3f3d149011224b))
* **ui:** collapsible sidebar groups + disambiguate duplicate group labels (shortest-unique path) ([36ec195](https://github.com/ysya/sshelter/commit/36ec195d565be69019988143286eb6fa5625c068))
* **ui:** command palette (⌘K) with connect/edit + terminal picker + per-row connect ([7d7ad65](https://github.com/ysya/sshelter/commit/7d7ad65aa8f6802ac19b32f7caeef29bbf2dd169))
* **ui:** config editor data layer, field-diff logic, app shell + host list ([ef06e51](https://github.com/ysya/sshelter/commit/ef06e51f40a69a416fda97ee00cd4152bc3aac52))
* **ui:** config intelligence panels + lint/discover/history dialogs ([17b09d9](https://github.com/ysya/sshelter/commit/17b09d9f1304614ac4b32dc8d8a357b1158693a4))
* **ui:** host editor, add/remove host, group/tags, drift banner ([4713c29](https://github.com/ysya/sshelter/commit/4713c2900dc8f9b373131456e701d5dbc0686f30))
* **ui:** native instrument-panel redesign — stacked editor, system fonts (mono for values), graphite + system-blue, source-list, refined chrome ([ed87420](https://github.com/ysya/sshelter/commit/ed87420d0b15a0446567aa86cc17a97755ec9f12))
* **ui:** native macOS desktop shell — fixed scroll regions, overlay titlebar, compact density, settings-style editor ([3cb1974](https://github.com/ysya/sshelter/commit/3cb197406776d85f68bffa097e66b3b3a140c62d))
* **ui:** refined terminal-shelter redesign + dark mode ([d230b9e](https://github.com/ysya/sshelter/commit/d230b9e18b751cc103461d81064c3e61fc2f3c96))
* **ui:** Settings sheet (⌘,) — move theme + default terminal out of the toolbar ([efbc506](https://github.com/ysya/sshelter/commit/efbc50623af52022c53476b15a02a34d8eb7807f))
* **ui:** sidebar Settings window — tray, close-to-tray, new-tab connect, config path, backup retention, discovery/drift/lint controls ([1697c82](https://github.com/ysya/sshelter/commit/1697c82eefcc7cb52bd7fe14c4329c471d40cdb8))
* **ui:** toolbar reload-from-disk button (manual refresh after external edits) ([30c3575](https://github.com/ysya/sshelter/commit/30c3575351c0c8f56ca335b1544849842249f207))
* **ui:** two-pane host editor — live ssh_config preview fills wide windows, stacks when narrow ([a5c76c3](https://github.com/ysya/sshelter/commit/a5c76c3dbfaf9a39f17c087b215bf5bb07b8f81d))
* wire TanStack Query + Zustand + app_platform IPC smoke ([29a8fb9](https://github.com/ysya/sshelter/commit/29a8fb947d36560938422e6c0c643724157ffc0d))


### Bug Fixes

* **config:** clamp backup retention to &gt;=1 (0 would prune the fresh backup) ([231b874](https://github.com/ysya/sshelter/commit/231b874cea7a6e3b2158c9ba7de327f4264d7b74))
* **config:** lint matches secondary host aliases, skips disabled directives + ProxyJump none; chain handles cross-host cycles ([5f66f19](https://github.com/ysya/sshelter/commit/5f66f19e97dbd11dcb4742c6a447d27ae512a590))
* **config:** preserve trailing whitespace in lexer split via trailing_ws field ([bfd5f1f](https://github.com/ysya/sshelter/commit/bfd5f1ff8f6a3bc71302dd1d315e788fa33ca33b))
* **config:** refuse to overwrite externally-changed files (drift conflict guard) ([235cf20](https://github.com/ysya/sshelter/commit/235cf20514f6abf090d233cdedd05c8f70c64c9d))
* **config:** skip unreadable Include files instead of aborting whole load ([f5510b5](https://github.com/ysya/sshelter/commit/f5510b585d4fc75390b2c483b64a576a7efc1183))
* **config:** strip newlines from values, insert new fields before trailing blank, doc host-disable asymmetry ([d602570](https://github.com/ysya/sshelter/commit/d602570902893da6cbfae56bcdb68a84c01df0f5))
* **connect:** reject leading-dash aliases (ssh argument-injection / -F config RCE) ([c58c733](https://github.com/ysya/sshelter/commit/c58c733903a8d46ef18039a1b79354de93796c54))
* **ui:** bump root font-size 13px→15px so body text lands at the intended ~13px (was rendering too small) ([b618d4c](https://github.com/ysya/sshelter/commit/b618d4c62346525b002bf4c5f0f9c68157d2d209))
* **ui:** hide number-input spinner steppers (type the value directly) ([482812c](https://github.com/ysya/sshelter/commit/482812c0f1b3e1364dd2dd24dfcb0197bdc87dd5))
* **ui:** host rows show resolved user@hostname as a distinct subtitle (no more duplicated alias) ([52c1292](https://github.com/ysya/sshelter/commit/52c129237454fb36a2685e63cffa9bedcb7735e7))
* **ui:** tri-state (yes/no/unset) selects so editing never silently deletes an explicit 'no' line ([4f8ecf3](https://github.com/ysya/sshelter/commit/4f8ecf387781a5770c3facaa79d7eb84daa1e85d))
* **window:** grant core:window start-dragging + toggle-maximize so the toolbar drags the window ([8a19dc0](https://github.com/ysya/sshelter/commit/8a19dc06651fa778de5662c5fa1c3ea71ed46d44))


### Miscellaneous Chores

* release 0.1.0 ([bc44ffa](https://github.com/ysya/sshelter/commit/bc44ffa6817fd9627d7fe3fc0d077b230a3fc2a2))
* release 0.2.0 ([237c744](https://github.com/ysya/sshelter/commit/237c744b72a0412bd007d1e4528400bfe363cae4))
