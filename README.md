import { GameMode, system, world } from "@minecraft/server";
import { Database } from "./utils/database";
import { PlayerName } from "./player";
import { ActionFormData, ModalFormData } from "@minecraft/server-ui";
import { ChestShop, ChestShopUI } from "./chestshop";
import { LandPos, LandPosXZ } from "./pos";
import { MessageFormData } from "@minecraft/server-ui";
import { config } from "./config";
import EventEmitter from "./events";
import { Crates, CratesUI } from "./crates";
const enteredEvents = new EventEmitter();
const playerLandPos = new Map();
const playerLandPos1 = new Map();
const playerLandPos2 = new Map();
// command.register("landpos1", "Set land pos1.", CommandPermissionLevel.NORMAL, (p, player) => {
//     PlayerLandPos.setPos1(player, player.location, player.dimension.id);
//     player.sendMessage(`[Land] §aSet pos 1`);
// }, {});
// command.register("landpos2", "Set land pos2.", CommandPermissionLevel.NORMAL, (p, player) => {
//     PlayerLandPos.setPos2(player, player.location, player.dimension.id);
//     player.sendMessage(`[Land] §aSet pos 2`);
// }, {});
// command.register("landclaim", "Claim land.", CommandPermissionLevel.NORMAL, (p, player) => {
//     const pos = PlayerLandPos.getLandPos(player);
//     if (!pos) {
//         player.sendMessage(`[Land] §cInvalid land pos`);
//         return;
//     }
//     LandMain.claimLand(pos, player, p.name);
// }, {
//     name: "string"
// });
// command.register("land", "Land Commands", CommandPermissionLevel.NORMAL, (p, player) => {
//     const landcmds = [ "landmembers" ]; 
//     const cmdslist = CustomCommandFactory.getHelpCommands(player).filter((v) => landcmds.includes(v[0].name)).map(([cmd, params]) => { `§r§e- §a${command.prefix()}${cmd.name} ${params.map((v) => `<${v[0]}: ${v[1]}>`).join(" ")} §b- ${cmd.description}` }).join("\n");
//     player.sendMessage(`§8---- §2Land§dCommands §8----§r\n§r§e- §a${command.prefix()}land <pos1|pos2|claim|delete|check> ${command.find("land") ? `§b- `+command.find("land")?.description+" " : ""}\n§r§e- §a${command.prefix()}landmember <add|remove|has> <player> ${command.find("landmember") ? `§b- `+command.find("landmember")?.description+" " : ""}\n${cmdslist}`);
// }, {
//     param: "string"
// });
export var LandUI;
(function (LandUI) {
    function main(player) {
        const posxz = LandPosXZ.create(player.location);
        const land = Land.getLand(posxz, player.dimension.id);
        if (land) {
            let isOwner = false;
            const form = new ActionFormData();
            form.title(`§2Land:§r ${land.name}`);
            form.body(`Name: ${land.name}§r\nOwner: §b${PlayerName.getName(land.owner)}§r\nPos: §a${LandPos.create(land.landpos.pos1, land.landpos.pos2, land.landpos.dimension).toString()}§r\nSize: §e${Land.getSize(land.landpos)} Blocks§r\n`);
            if (land.owner === player.id) {
                form.button(`§2Set Name`, `textures/items/name_tag.png`);
                form.button(`§dSet Owner`, `textures/ui/permissions_op_crown.png`);
                form.button(`§eMembers`, `textures/ui/multiplayer_glyph_color.png`);
                form.button(`§cSell Land`, `textures/ui/icon_new.png`);
                isOwner = true;
            }
            form.button(`§7[ §cCLOSE §7]`, `textures/blocks/barrier.png`);
            form.show(player).then((res) => {
                if (res.canceled || res.selection === undefined)
                    return;
                if (isOwner) {
                    if (res.selection === 0)
                        setName(player, land);
                    if (res.selection === 1)
                        setOwner(player, land);
                    if (res.selection === 2)
                        landMembers(player, land);
                    if (res.selection === 3)
                        landSellConfirm(player, land);
                    return;
                }
            });
        }
        else {
            const form = new ActionFormData();
            const pos1 = PlayerLandPos.getPos1(player);
            const pos2 = PlayerLandPos.getPos2(player);
            const landpos = PlayerLandPos.getLandPos(player);
            let isSelected = false;
            form.title(`§2Land§dClaim`);
            form.body(`§bQ: §aHow to claim land?\n§bA: §aUse any item and rename to §rClaim Wand§a for pointer§r\n\n§e- §2Click Block: §dSet Position 1\n§e- §2Break Block: §dSet Position 2\n\n${landpos ? `§rPos 1: §a${pos1 ? pos1[0].toString() + ": " + pos1[1] : "Not Set"}§r\nPos 2: §a${pos2 ? pos2[0].toString() + ": " + pos2[1] : "Not Set"}§r\nLand Pos: §a${landpos ? landpos.toString() : "Not Set"}§r\nSize: §b${Land.getSize(landpos)} Blocks§r\nPrice: §e${config.currency}${landpos ? Land.getLandPrice(landpos) : 0}§r\nCan Claim: §a${landpos ? !Land.hasClaimedV2(landpos) : "Not Set"}` : `§rNo Selection`}`);
            if (landpos) {
                form.button(`§2Claim Land`);
                form.button(`§2Delete Pos`);
                isSelected = true;
            }
            form.button(`§7[ §cCLOSE §7]`, `textures/blocks/barrier.png`);
            form.show(player).then((res) => {
                if (res.canceled || res.selection === undefined)
                    return;
                if (isSelected) {
                    if (res.selection === 2)
                        return;
                    if (!landpos) {
                        player.sendMessage(`[Land] §cSelection not found!`);
                        return;
                    }
                    if (res.selection === 0) {
                        LandMain.claimLand(landpos, player);
                        PlayerLandPos.deletePos(player);
                    }
                    if (res.selection === 1)
                        PlayerLandPos.deletePos(player);
                    return;
                }
            });
        }
    }
    LandUI.main = main;
    function setName(player, land) {
        const form = new ModalFormData();
        form.title(`§aSetName: §r${land.name}`);
        form.textField(`Name`, `max 16 characters`);
        form.show(player).then((res) => {
            if (res.canceled || res.formValues === undefined)
                return;
            LandMain.setLandName(land.landpos.pos1, land.landpos.dimension, player, res.formValues[0].toString());
        });
    }
    LandUI.setName = setName;
    function setOwner(player, land) {
        const form = new ActionFormData();
        form.title(`§aSetOwner: §r${land.name}`);
        form.body(`Change land owner`);
        const players = world.getPlayers().filter((v) => v.id !== player.id);
        for (const owner of players) {
            form.button(`§d${owner.name}§r\n§eClick to set Owner`, `textures/ui/permissions_op_crown.png`);
        }
        form.button(`§8[ §cBACK §8]`, `textures/blocks/barrier.png`);
        form.show(player).then((res) => {
            if (res.canceled || res.selection === undefined)
                return;
            if (res.selection === players.length) {
                main(player);
                return;
            }
            setOwnerConfirm(player, players[res.selection], land);
        });
    }
    LandUI.setOwner = setOwner;
    function setOwnerConfirm(player, target, land) {
        const form = new MessageFormData();
        form.title(`§aSetOwner: ${land.name}`);
        form.body(`Are tou sure wanna change this land owner to §b${target.name}§r?\n\n§rLand: ${land.name}§r\nPos: §a${land.landpos.toString()}§r\nSize: §a${Land.getSize(land.landpos)} Blocks§r`);
        form.button1("§8[ §cCANCEL §8]");
        form.button2("§l§2CONFIRM");
        form.show(player).then((res) => {
            if (res.canceled || res.selection === undefined)
                return;
            if (res.selection === 1)
                LandMain.setOwner(land.landpos.pos1, land.landpos.dimension, player, target);
        });
    }
    LandUI.setOwnerConfirm = setOwnerConfirm;
    function landSellConfirm(player, land) {
        const form = new MessageFormData();
        form.title(`§aSellLand: ${land.name}`);
        form.body(`Are tou sure wanna sell this land?\n\n§rLand: ${land.name}§r\nPos: §a${LandPos.create(land.landpos.pos1, land.landpos.pos2, land.landpos.dimension).toString()}§r\nSize: §a${Land.getSize(land.landpos)} Blocks§r\nSell: §e${config.currency}${Land.getLandSell(land.landpos)}`);
        form.button1("§8[ §cCANCEL §8]");
        form.button2("§l§cCONFIRM");
        form.show(player).then((res) => {
            if (res.canceled || res.selection === undefined)
                return;
            if (res.selection === 1)
                LandMain.deleteLand(land.landpos.pos1, land.landpos.dimension, player);
        });
    }
    LandUI.landSellConfirm = landSellConfirm;
    function landMembers(player, land) {
        const form = new ActionFormData();
        form.title(`§aMembers: §r${land.name}`);
        form.body(`Click to remove member from this land`);
        const members = land.members;
        for (const member of members) {
            form.button(`§b${PlayerName.getName(member)}§r\n§cClick to remove`, `textures/ui/icon_agent.png`);
        }
        form.button(`§2Add Member`, `textures/ui/multiplayer_glyph_color.png`);
        form.button(`§8[ §cBACK §8]`, `textures/blocks/barrier.png`);
        form.show(player).then((res) => {
            if (res.canceled || res.selection === undefined)
                return;
            if (res.selection === members.length) {
                addMember(player, land);
                return;
            }
            if (res.selection === members.length + 1) {
                main(player);
                return;
            }
            LandMain.removeMemberByXuid(land.landpos.pos1, land.landpos.dimension, player, members[res.selection]);
        });
    }
    LandUI.landMembers = landMembers;
    function addMember(player, land) {
        const form = new ActionFormData();
        form.title(`§aAddMember: §r${land.name}`);
        form.body(`Click to add member to this land`);
        const players = world.getPlayers().filter((v) => v.id !== player.id && !LandMain.isMember(land.landpos.pos1, land.landpos.dimension, v));
        for (const member of players) {
            form.button(`§b${member.name}§r\n§aClick to add`, `textures/ui/icon_agent.png`);
        }
        form.button(`§8[ §cBACK §8]`, `textures/blocks/barrier.png`);
        form.show(player).then((res) => {
            if (res.canceled || res.selection === undefined)
                return;
            if (res.selection === players.length) {
                landMembers(player, land);
                return;
            }
            LandMain.addMember(land.landpos.pos1, land.landpos.dimension, player, players[res.selection]);
        });
    }
    LandUI.addMember = addMember;
    function admin(player) {
        const form = new ActionFormData();
        form.title(`§2Admin§dLand`);
        form.body(`Price: §a${config.currency}${Land.getPrice()}/Block§r\nSell: §a${config.currency}${Land.getSell()}/Block§r\nMinimum Land Size: §d${Land.getMin()} Blocks§r\nMaximum Land Size: §d${Land.getMax()} Blocks§r`);
        form.button(`§2Set Price`);
        form.button(`§2Set Sell`);
        form.button(`§2Set Minimum Land Size`);
        form.button(`§2Set Maximum Land Size`);
        form.show(player).then((res) => {
            if (res.canceled || res.selection === undefined)
                return;
            if (res.selection === 0)
                setPrice(player);
            if (res.selection === 1)
                setSell(player);
            if (res.selection === 2)
                setMin(player);
            if (res.selection === 3)
                setMax(player);
        });
    }
    LandUI.admin = admin;
    function setPrice(player) {
        const form = new ModalFormData();
        form.title(`§2Land: §eSetPrice`);
        form.textField(`Value`, Land.getPrice().toString());
        form.show(player).then((res) => {
            if (res.canceled || res.formValues === undefined)
                return;
            if (Number.isNaN(res.formValues[0])) {
                player.sendMessage(`[Land] §cInvalid value!`);
                return;
            }
            Land.setLandPrice(Number(res.formValues[0]) * 10 / 10);
        });
    }
    LandUI.setPrice = setPrice;
    function setSell(player) {
        const form = new ModalFormData();
        form.title(`§2Land: §eSetSell`);
        form.textField(`Value`, Land.getSell().toString());
        form.show(player).then((res) => {
            if (res.canceled || res.formValues === undefined)
                return;
            if (Number.isNaN(res.formValues[0])) {
                player.sendMessage(`[Land] §cInvalid value!`);
                return;
            }
            Land.setLandSell(Number(res.formValues[0]) * 10 / 10);
        });
    }
    LandUI.setSell = setSell;
    function setMin(player) {
        const form = new ModalFormData();
        form.title(`§2Land: §eSetMin`);
        form.textField(`Value`, Land.getMin().toString());
        form.show(player).then((res) => {
            if (res.canceled || res.formValues === undefined)
                return;
            if (Number.isNaN(res.formValues[0])) {
                player.sendMessage(`[Land] §cInvalid value!`);
                return;
            }
            Land.setMin(Number(res.formValues[0]) * 10 / 10);
        });
    }
    LandUI.setMin = setMin;
    function setMax(player) {
        const form = new ModalFormData();
        form.title(`§2Land: §eSetMax`);
        form.textField(`Value`, Land.getMax().toString());
        form.show(player).then((res) => {
            if (res.canceled || res.formValues === undefined)
                return;
            if (Number.isNaN(res.formValues[0])) {
                player.sendMessage(`[Land] §cInvalid value!`);
                return;
            }
            Land.setMax(Number(res.formValues[0]) * 10 / 10);
        });
    }
    LandUI.setMax = setMax;
})(LandUI || (LandUI = {}));
export var PlayerLandPos;
(function (PlayerLandPos) {
    function getPos1(player) {
        var _a;
        return (_a = playerLandPos1.get(player)) !== null && _a !== void 0 ? _a : null;
    }
    PlayerLandPos.getPos1 = getPos1;
    function getPos2(player) {
        var _a;
        return (_a = playerLandPos2.get(player)) !== null && _a !== void 0 ? _a : null;
    }
    PlayerLandPos.getPos2 = getPos2;
    function setPos1(player, pos, dimensionId) {
        playerLandPos1.set(player, [LandPosXZ.create(pos), dimensionId]);
    }
    PlayerLandPos.setPos1 = setPos1;
    function setPos2(player, pos, dimensionId) {
        playerLandPos2.set(player, [LandPosXZ.create(pos), dimensionId]);
    }
    PlayerLandPos.setPos2 = setPos2;
    function getLandPos(player) {
        const pos1 = getPos1(player);
        const pos2 = getPos2(player);
        if (!pos1 || !pos2)
            return null;
        if (pos1[1] !== pos2[1])
            return null;
        const landPos = LandPos.create(pos1[0], pos2[0], pos1[1]);
        return landPos;
    }
    PlayerLandPos.getLandPos = getLandPos;
    function deletePos(player) {
        playerLandPos1.delete(player);
        playerLandPos2.delete(player);
    }
    PlayerLandPos.deletePos = deletePos;
})(PlayerLandPos || (PlayerLandPos = {}));
/**Main of plugin */
export var Land;
(function (Land) {
    /**Get price perblock */
    function getPrice() {
        const data = Database.get("candra:landconfig");
        if (typeof data === "undefined")
            return 3;
        if (data.price === undefined || data.price < 0 || data.price === 0)
            return 0;
        else
            return data.price;
    }
    Land.getPrice = getPrice;
    /**Get sell perblock */
    function getSell() {
        const data = Database.get("candra:landconfig");
        if (typeof data === "undefined")
            return 1.5;
        if (data.sell === undefined || data.sell < 0 || data.sell === 0)
            return 0;
        else
            return data.sell;
    }
    Land.getSell = getSell;
    /**Get land price */
    function getLandPrice(landpos) {
        const size = getSize(landpos);
        if (size <= 0)
            return 0;
        else
            return ((size * getPrice()) * 10 / 10);
    }
    Land.getLandPrice = getLandPrice;
    /**Get land sell */
    function getLandSell(landpos) {
        const size = getSize(landpos);
        if (size <= 0)
            return 0;
        else
            return ((size * getSell()) * 10 / 10);
    }
    Land.getLandSell = getLandSell;
    /**Get max land size */
    function getMax() {
        const data = Database.get("candra:landconfig");
        if (typeof data === "undefined")
            return 2000;
        if (data.land_size.max < 1)
            return 1;
        else
            return data.land_size.max;
    }
    Land.getMax = getMax;
    /**Get min land size */
    function getMin() {
        const data = Database.get("candra:landconfig");
        if (typeof data === "undefined")
            return 9;
        if (data.land_size.min < 1)
            return 1;
        else
            return data.land_size.min;
    }
    Land.getMin = getMin;
    /**Set max land size */
    function setMax(value) {
        if (value < getMin())
            return false;
        let data = Database.get("candra:landconfig");
        if (typeof data === "undefined")
            data = {
                price: getPrice(),
                sell: getSell(),
                land_size: {
                    min: getMin(),
                    max: getMax(),
                },
            };
        data.land_size.max = value;
        Database.set("candra:landconfig", data);
        return true;
    }
    Land.setMax = setMax;
    /**Set min land size */
    function setLandPrice(value) {
        if (value < 0)
            return false;
        let data = Database.get("candra:landconfig");
        if (typeof data === "undefined")
            data = {
                price: getPrice(),
                sell: getSell(),
                land_size: {
                    min: getMin(),
                    max: getMax(),
                },
            };
        data.price = value;
        Database.set("candra:landconfig", data);
        return true;
    }
    Land.setLandPrice = setLandPrice;
    /**Set max land size */
    function setLandSell(value) {
        if (value < 0)
            return false;
        let data = Database.get("candra:landconfig");
        if (typeof data === "undefined")
            data = {
                price: getPrice(),
                sell: getSell(),
                land_size: {
                    min: getMin(),
                    max: getMax(),
                },
            };
        data.sell = value;
        Database.set("candra:landconfig", data);
        return true;
    }
    Land.setLandSell = setLandSell;
    /**Set min land size */
    function setMin(value) {
        if (value < 1)
            return false;
        let data = Database.get("candra:landconfig");
        if (typeof data === "undefined")
            data = {
                price: getPrice(),
                sell: getSell(),
                land_size: {
                    min: getMin(),
                    max: getMax(),
                },
            };
        data.land_size.min = value;
        Database.set("candra:landconfig", data);
        return true;
    }
    Land.setMin = setMin;
    /**Get all lands */
    function getLands() {
        let lands = [];
        const data = Database.get("candra:lands");
        if (data === undefined)
            return lands;
        lands = data;
        return lands;
    }
    Land.getLands = getLands;
    /**Check land has claimed or no */
    function hasClaimed(pos, dimensionId) {
        let result = false;
        for (const land of getLands()) {
            const landpos = land.landpos;
            if (landpos.pos1.x >= pos.x && landpos.pos1.z >= pos.z && landpos.pos2.x <= pos.x && landpos.pos2.z <= pos.z && landpos.dimension === dimensionId)
                result = true;
            else if (landpos.pos1.x <= pos.x && landpos.pos1.z <= pos.z && landpos.pos2.x >= pos.x && landpos.pos2.z >= pos.z && landpos.dimension === dimensionId)
                result = true;
        }
        return result;
    }
    Land.hasClaimed = hasClaimed;
    /**Check land has claimed or no by LandPos */
    function hasClaimedV2(pos) {
        let result = false;
        for (const land of getLands()) {
            const landpos = land.landpos;
            if (landpos.pos1.x >= pos.pos1.x && landpos.pos1.z >= pos.pos1.z && landpos.pos2.x <= pos.pos2.x && landpos.pos2.z <= pos.pos2.z && landpos.dimension === pos.dimension)
                result = true;
            else if (landpos.pos1.x <= pos.pos1.x && landpos.pos1.z <= pos.pos1.z && landpos.pos2.x >= pos.pos2.x && landpos.pos2.z >= pos.pos2.z && landpos.dimension === pos.dimension)
                result = true;
        }
        return result;
    }
    Land.hasClaimedV2 = hasClaimedV2;
    /**Get land size. */
    function getSize(pos) {
        let calculate = (Math.abs(pos.pos2.x - pos.pos1.x) * Math.abs(pos.pos2.z - pos.pos1.z));
        return calculate;
    }
    Land.getSize = getSize;
    /**Get land data from position. */
    function getLand(pos, dimensionId) {
        let result = undefined;
        for (const land of getLands()) {
            const landpos = land.landpos;
            if (landpos.pos1.x >= pos.x && landpos.pos1.z >= pos.z && landpos.pos2.x <= pos.x && landpos.pos2.z <= pos.z && landpos.dimension === dimensionId)
                result = land;
            else if (landpos.pos1.x <= pos.x && landpos.pos1.z <= pos.z && landpos.pos2.x >= pos.x && landpos.pos2.z >= pos.z && landpos.dimension === dimensionId)
                result = land;
        }
        return result;
    }
    Land.getLand = getLand;
    function update() {
        const data = Database.get("candra:landconfig");
        if (typeof data === "undefined") {
            const landConfig = {
                price: 3,
                sell: 1.5,
                land_size: {
                    min: 9,
                    max: 2000,
                },
            };
            Database.set("candra:landconfig", landConfig);
        }
    }
    Land.update = update;
    function updateEvents() {
        for (const player of world.getPlayers()) {
            const land = Land.getLand(LandPosXZ.create(player.location), player.dimension.id);
            if (land) {
                const landpos = LandPos.create(land.landpos.pos1, land.landpos.pos2, land.landpos.dimension);
                const check = playerLandPos.get(player);
                if (!check || !Boolean(check.pos1.x === landpos.pos1.x && check.pos1.z === landpos.pos1.z && check.pos2.x === landpos.pos2.x && check.pos2.z === landpos.pos2.z && check.dimension === landpos.dimension)) {
                    const data = { player, land };
                    playerLandPos.set(player, landpos);
                    enteredEvents.emit("land:enteredland", data);
                }
            }
            if (!land && playerLandPos.get(player)) {
                const data = { player };
                playerLandPos.delete(player);
                enteredEvents.emit("land:enteredwild", data);
            }
        }
    }
    Land.updateEvents = updateEvents;
    let events;
    (function (events) {
        function enteredLand(callback) {
            enteredEvents.on("land:enteredland", callback);
        }
        events.enteredLand = enteredLand;
        function enteredWild(callback) {
            enteredEvents.on("land:enteredwild", callback);
        }
        events.enteredWild = enteredWild;
    })(events = Land.events || (Land.events = {}));
})(Land || (Land = {}));
/**Main of Land */
export var LandMain;
(function (LandMain) {
    /**Claim a new land */
    function claimLand(landPos, player, name) {
        if (Land.hasClaimedV2(landPos)) {
            player.sendMessage("[Land] §cLand has been claimed!");
            return false;
        }
        if (Land.getSize(landPos) > Land.getMax() || Land.getSize(landPos) < Land.getMin()) {
            player.sendMessage("[Land] §cInvalid land size!");
            return false;
        }
        const objective = world.scoreboard.getObjective(config.objective);
        if (!objective) {
            player.sendMessage(`[Land] §cYou dont have enough §e${config.objective}§r§c to buy a llan!`);
            return false;
        }
        const score = objective.getScore(player);
        if (!score || score < Land.getLandPrice(landPos)) {
            player.sendMessage(`[Land] §cYou dont have enough §e${config.objective}§r§c to buy a llan!`);
            return false;
        }
        if (name)
            name = name.substring(0, 16);
        let land = {
            owner: player.id,
            // player_options: {
            //     Explosion: false,
            //     SplashPotion: false,
            //     PVP: false,
            //     OpenChest: false,
            //     AttackMob: false,
            //     PressButton: false,
            //     UseBlock: false,
            //     UseDoor: false,
            //     PlayerInteract: false,
            // },
            // member_options: {
            //     Explosion: false,
            //     SplashPotion: false,
            //     PVP: false,
            //     OpenChest: true,
            //     AttackMob: true,
            //     PressButton: true,
            //     UseBlock: true,
            //     UseDoor: true,
            //     PlayerInteract: true,
            // },
            members: [],
            name: name !== null && name !== void 0 ? name : `${player.name} Land`,
            landpos: landPos,
        };
        player.sendMessage(`[Land] §aSuccess claim this land §r${land.landpos.pos1.toString()}, ${land.landpos.pos2.toString()} §r§afor §e${config.currency}${Land.getLandPrice(landPos)}`);
        objective.setScore(player, score - Land.getLandPrice(landPos));
        let lands = Land.getLands();
        lands.push(land);
        Database.set("candra:lands", lands);
        return true;
    }
    LandMain.claimLand = claimLand;
    /**Remove land */
    function deleteLand(pos, dimensionId, player) {
        const land = Land.getLand(pos, dimensionId);
        if (!land) {
            player.sendMessage("[Land] §cLand not found!");
            return false;
        }
        if (land.owner !== player.id) {
            player.sendMessage("[Land] §cYou don't have permissions!");
            return false;
        }
        const objective = world.scoreboard.getObjective(config.objective);
        if (!objective) {
            player.sendMessage(`[Land] §cYou dont have enough §e${config.objective}§r§c to buy a clan!`);
            return false;
        }
        const score = objective.getScore(player);
        if (!score || score < Land.getLandPrice(land.landpos)) {
            player.sendMessage(`[Land] §cYou dont have enough §e${config.objective}§r§c to buy a clan!`);
            return false;
        }
        const landpos = LandPos.create(land.landpos.pos1, land.landpos.pos2, land.landpos.dimension);
        player.sendMessage(`[Land] §aSuccess remove this land §r${landpos.toString()} §r§afor §e${config.currency}${Land.getLandSell(land.landpos)}`);
        objective.setScore(player, score + Land.getLandSell(landpos));
        let lands = Land.getLands();
        let newLands = lands.filter((v, i) => i !== getLandIndex(landpos.pos1, landpos.dimension));
        Database.set("candra:lands", newLands);
        return true;
    }
    LandMain.deleteLand = deleteLand;
    /**Set land owner */
    function setOwner(pos, dimensionId, player, owner) {
        const land = Land.getLand(pos, dimensionId);
        if (!land) {
            player.sendMessage("[Land] §cLand not found!");
            return false;
        }
        if (land.owner !== player.id) {
            player.sendMessage("[Land] §cYou don't have permissions!");
            return false;
        }
        if (land.owner === owner.id) {
            player.sendMessage("[Land] §cWTF U DOING?!");
            return false;
        }
        let lands = Land.getLands();
        if (land.members.includes(owner.id)) {
            lands[getLandIndex(pos, dimensionId)].members = lands[getLandIndex(pos, dimensionId)].members.filter((v) => v !== owner.id);
        }
        player.sendMessage(`[Land] §e${owner.name} §ahas been set as owner in§r ${land.name}§r ${land.landpos.toString()}`);
        owner.sendMessage(`[Land] §e${player.name} §ahas set you as owner in§r ${land.name}§r ${land.landpos.toString()}`);
        lands[getLandIndex(pos, dimensionId)].owner = owner.id;
        Database.set("candra:lands", lands);
        return true;
    }
    LandMain.setOwner = setOwner;
    /**Set land owner by Unique identifier */
    function setOwnerById(pos, dimensionId, player, owner) {
        var _a;
        const land = Land.getLand(pos, dimensionId);
        if (!land) {
            player.sendMessage("[Land] §cLand not found!");
            return false;
        }
        if (land.owner !== player.id) {
            player.sendMessage("[Land] §cYou don't have permissions!");
            return false;
        }
        if (land.owner === owner) {
            player.sendMessage("[Land] §cWTF U DOING?!");
            return false;
        }
        let lands = Land.getLands();
        if (land.members.includes(owner)) {
            lands[getLandIndex(pos, dimensionId)].members = lands[getLandIndex(pos, dimensionId)].members.filter((v) => v !== owner);
        }
        const _owner = PlayerName.getName(owner);
        player.sendMessage(`[Land] §e${_owner} §ahas been set as owner in§r ${land.name}§r ${land.landpos.toString()}`);
        (_a = world.getPlayers().find((v) => v.id === owner)) === null || _a === void 0 ? void 0 : _a.sendMessage(`[Land] §e${player.name} §ahas set you as owner in§r ${land.name}§r ${land.landpos.toString()}`);
        lands[getLandIndex(pos, dimensionId)].owner = owner;
        Database.set("candra:lands", lands);
        return true;
    }
    LandMain.setOwnerById = setOwnerById;
    /**Check this is member */
    function isMember(pos, dimensionId, player) {
        const land = Land.getLand(pos, dimensionId);
        if (!land)
            return false;
        else
            return land.members.includes(player.id);
    }
    LandMain.isMember = isMember;
    /**Get all members */
    function getMembers(pos, dimensionId) {
        const land = Land.getLand(pos, dimensionId);
        if (!land)
            return null;
        else
            return land.members;
    }
    LandMain.getMembers = getMembers;
    /**Add new member */
    function addMember(pos, dimensionId, player, member) {
        var _a;
        const land = Land.getLand(pos, dimensionId);
        if (!land) {
            player.sendMessage("[Land] §cLand not found!");
            return false;
        }
        if (land.owner !== player.id) {
            player.sendMessage("[Land] §cYou don't have permissions!");
            return false;
        }
        if (land.owner === member.id) {
            player.sendMessage("[Land] §cWTF U DOING?!");
            return false;
        }
        if (land.members.includes(member.id)) {
            player.sendMessage("[Land] §cThis player has been added to member before");
            return false;
        }
        player.sendMessage(`[Land] §e${member.name} §ahas been added as member in§r ${land.name}§r ${land.landpos.toString()}`);
        member.sendMessage(`[Land] §e${player.name} §ahas added you as member in§r ${land.name}§r ${land.landpos.toString()}`);
        let lands = Land.getLands();
        let members = (_a = getMembers(pos, dimensionId)) !== null && _a !== void 0 ? _a : [];
        members.push(member.id);
        lands[getLandIndex(pos, dimensionId)].members = members;
        Database.set("candra:lands", lands);
        return true;
    }
    LandMain.addMember = addMember;
    /**Remove a member */
    function removeMember(pos, dimensionId, player, member) {
        const land = Land.getLand(pos, dimensionId);
        if (!land) {
            player.sendMessage("[Land] §cLand not found!");
            return false;
        }
        if (land.owner !== player.id) {
            player.sendMessage("[Land] §cYou don't have permissions!");
            return false;
        }
        if (!land.members.includes(member.id)) {
            player.sendMessage("[Land] §cMember not found!");
            return false;
        }
        player.sendMessage(`[Land] §e${member.name} §ahas been removed from members in§r ${land.name}§r ${land.landpos.toString()}`);
        member.sendMessage(`[Land] §e${player.name} §ahas remove you from members in§r ${land.name}§r ${land.landpos.toString()}`);
        let lands = Land.getLands();
        lands[getLandIndex(pos, dimensionId)].members = lands[getLandIndex(pos, dimensionId)].members.filter((v) => v !== member.id);
        Database.set("candra:lands", lands);
        return true;
    }
    LandMain.removeMember = removeMember;
    /**Add new member by xuid */
    function addMemberByXuid(pos, dimensionId, player, member) {
        var _a;
        const land = Land.getLand(pos, dimensionId);
        if (!land) {
            player.sendMessage("[Land] §cLand not found!");
            return false;
        }
        if (land.owner !== player.id) {
            player.sendMessage("[Land] §cYou don't have permissions!");
            return false;
        }
        if (land.owner === member) {
            player.sendMessage("[Land] §cWTF U DOING?!");
            return false;
        }
        if (land.members.includes(member)) {
            player.sendMessage("[Land] §cThis player has been added to member before");
            return false;
        }
        const _member = PlayerName.getName(member);
        player.sendMessage(`[Land] §e${_member} §ahas been added as member in§r ${land.name}§r ${land.landpos.toString()}`);
        (_a = world.getPlayers().find((v) => v.id === member)) === null || _a === void 0 ? void 0 : _a.sendMessage(`[Land] §e${player.name} §ahas added you as member in§r ${land.name}§r ${land.landpos.toString()}`);
        let lands = Land.getLands();
        lands[getLandIndex(pos, dimensionId)].members.push(member);
        Database.set("candra:lands", lands);
        return true;
    }
    LandMain.addMemberByXuid = addMemberByXuid;
    /**Remove a member by xuid */
    function removeMemberByXuid(pos, dimensionId, player, member) {
        var _a;
        const land = Land.getLand(pos, dimensionId);
        if (!land) {
            player.sendMessage("[Land] §cLand not found!");
            return false;
        }
        if (land.owner !== player.id) {
            player.sendMessage("[Land] §cYou don't have permissions!");
            return false;
        }
        if (!land.members.includes(member)) {
            player.sendMessage("[Land] §cMember not found!");
            return false;
        }
        const _member = PlayerName.getName(member);
        player.sendMessage(`[Land] §e${_member} §ahas been removed from members in§r ${land.name}§r ${land.landpos.toString()}`);
        (_a = world.getPlayers().find((v) => v.id === member)) === null || _a === void 0 ? void 0 : _a.sendMessage(`[Land] §e${player.name} §ahas remove you from members in§r ${land.name}§r ${land.landpos.toString()}`);
        let lands = Land.getLands();
        lands[getLandIndex(pos, dimensionId)].members = lands[getLandIndex(pos, dimensionId)].members.filter((v) => v !== member);
        Database.set("candra:lands", lands);
        return true;
    }
    LandMain.removeMemberByXuid = removeMemberByXuid;
    /**Set land name */
    function setLandName(pos, dimensionId, player, name) {
        const land = Land.getLand(pos, dimensionId);
        if (!land) {
            player.sendMessage("[Land] §cLand not found!");
            return false;
        }
        if (land.owner !== player.id) {
            player.sendMessage("[Land] §cYou don't have permissions!");
            return false;
        }
        if (name === "" || name === " ".repeat(name.length) || name.length > 16) {
            player.sendMessage("player.invalid.name");
            return false;
        }
        player.sendMessage(`[Land] §aSuccess change land name to §r${name}`);
        let lands = Land.getLands();
        lands[getLandIndex(pos, dimensionId)].name = name;
        Database.set("candra:lands", lands);
        return true;
    }
    LandMain.setLandName = setLandName;
    // /**Get land options */
    // export function getLandOptions(pos: LandPosXZ, dimensionId: string): PlayerLandOptions|null {
    //     const land = Land.getLand(pos, dimensionId);
    //     if (land) return land.player_options;
    //     else return null;
    // }
    // /**Get land option */
    // export function getLandOption(pos: LandPosXZ, dimensionId: string, option: keyof PlayerLandOptions): boolean|null {
    //     const land = Land.getLand(pos, dimensionId);
    //     if (!land) return null;
    //     else return land.player_options[option].valueOf();
    // }
    // /**Set land options */
    // export function setLandOptions(player: Player, pos: LandPosXZ, dimensionId: string, option: keyof PlayerLandOptions, value: boolean): boolean {
    //     const idx = getLandIndex(pos, dimensionId);
    //     if (idx === -1) return false;
    //     if (!isOwner(pos, dimensionId, player)) {
    //         player.sendMessage("[Land] §cYou don't have permissions!");
    //         return false;
    //     }
    //     let lands = Land.getLands();
    //     lands[idx].player_options[option]=value;
    //     Database.set("candra:lands", lands);
    //     return true;
    // }
    // /**Get land member options */
    // export function getMemberOptions(pos: LandPosXZ, dimensionId: string): PlayerLandOptions|null {
    //     const land = Land.getLand(pos, dimensionId);
    //     if (land) return land.member_options;
    //     else return null;
    // }
    // /**Get land member option */
    // export function getMemberOption(pos: LandPosXZ, dimensionId: string, option: keyof PlayerLandOptions): boolean|null {
    //     const land = Land.getLand(pos, dimensionId);
    //     if (!land) return null;
    //     else return land.member_options[option].valueOf();
    // }
    // /**Set land member options */
    // export function setMemberOptions(player: Player, pos: LandPosXZ, dimensionId: string, option: keyof PlayerLandOptions, value: boolean): boolean {
    //     const idx = getLandIndex(pos, dimensionId);
    //     if (idx === -1) return false;
    //     if (!isOwner(pos, dimensionId, player)) {
    //         player.sendMessage("[Land] §cYou don't have permissions!");
    //         return false;
    //     }
    //     let lands = Land.getLands();
    //     lands[idx].member_options[option]=value;
    //     Database.set("candra:lands", lands);
    //     return true;
    // }
    function isOwner(pos, dimensionId, player) {
        const land = Land.getLand(pos, dimensionId);
        if (!land)
            return false;
        else
            return (land.owner === player.id);
    }
    LandMain.isOwner = isOwner;
    function getLandIndex(pos, dimensionId) {
        let result = -1;
        for (const [i, land] of Land.getLands().entries()) {
            const landpos = land.landpos;
            if (landpos.pos1.x >= pos.x && landpos.pos1.z >= pos.z && landpos.pos2.x <= pos.x && landpos.pos2.z <= pos.z && landpos.dimension === dimensionId)
                result = i;
            else if (landpos.pos1.x <= pos.x && landpos.pos1.z <= pos.z && landpos.pos2.x >= pos.x && landpos.pos2.z >= pos.z && landpos.dimension === dimensionId)
                result = i;
        }
        return result;
    }
})(LandMain || (LandMain = {}));
function protect(player, position = player.location) {
    const pos = LandPosXZ.create(position);
    const land = Land.getLand(pos, player.dimension.id);
    if (!land)
        return false;
    if (player.matches({ gameMode: GameMode.creative }) && player.isOp() || player.matches({ gameMode: GameMode.creative }) && player.hasTag("navperm:admin"))
        return false;
    if (LandMain.isOwner(pos, player.dimension.id, player))
        return false;
    if (LandMain.isMember(pos, player.dimension.id, player))
        return false;
    return true;
}
function protect2(position, dimensionId) {
    const pos = LandPosXZ.create(position);
    const land = Land.getLand(pos, dimensionId);
    if (!land)
        return false;
    return true;
}
Land.events.enteredLand((data) => {
    data.player.sendMessage(`§eEntered §r${data.land.name}`);
});
Land.events.enteredWild((data) => {
    data.player.sendMessage(`§eEntered §2Wild`);
});
system.runInterval(() => {
    system.run(() => {
        Land.update();
        Land.updateEvents();
    });
});
world.beforeEvents.explosion.subscribe((data) => {
    const affectedBlocks = data.getImpactedBlocks().filter((block) => {
        const pos = {
            x: Math.floor(block.location.x),
            y: Math.floor(block.location.y),
            z: Math.floor(block.location.z),
        };
        const dimensionId = block.dimension.id;
        if (ChestShop.getShop(pos, dimensionId))
            return false;
        return !protect2(pos, dimensionId);
    });
    data.setImpactedBlocks(affectedBlocks);
});
world.beforeEvents.playerBreakBlock.subscribe((data) => {
    var _a;
    data.cancel = protect(data.player, data.block.location);
    const crate = Crates.getCrate(data.block.location, data.dimension.id);
    if (crate) {
        data.cancel = true;
        if (data.player.isOp() || data.player.hasTag("navperm:admin"))
            system.run(() => {
                CratesUI.deleteCrate(data.player, data.block.location, data.dimension.id);
            });
    }
    const shop = ChestShop.getShop(data.block.location, data.block.dimension.id);
    if (shop && !shop.isOwner(data.player)) {
        data.player.sendMessage(`[CShop] §cYou don't have access to destroy this`);
        data.cancel = true;
    }
    if (!protect(data.player, data.block.location) && data.itemStack && ((_a = data.itemStack.nameTag) === null || _a === void 0 ? void 0 : _a.toLowerCase()) === "claim wand") {
        PlayerLandPos.setPos2(data.player, data.block.location, data.block.dimension.id);
        data.player.sendMessage(`[Land] §aSet 2`);
        data.cancel = true;
    }
    if (shop && shop.isOwner(data.player))
        data.cancel = false;
});
world.beforeEvents.playerInteractWithBlock.subscribe((data) => {
    var _a, _b, _c, _d, _e, _f, _g;
    data.cancel = protect(data.player, data.block.location);
    if (Crates.isShulkerbox(data.block)) {
        const pos = {
            x: Math.floor(data.block.location.x),
            y: Math.floor(data.block.location.y),
            z: Math.floor(data.block.location.z),
        };
        const dimensionId = data.block.dimension.id;
        const crate = Crates.getCrate(pos, dimensionId);
        if (crate) {
            if (data.player.isOp() && data.player.isSneaking || data.player.hasTag("navperm:admin") && data.player.isSneaking)
                return;
            else {
                data.cancel = true;
                system.run(() => {
                    if (data.player.isSneaking && data.itemStack === undefined)
                        CratesUI.crateItems(data.player, pos, dimensionId);
                    else if (!data.player.isSneaking)
                        Crates.openCrate(data.player, pos, dimensionId);
                });
            }
        }
        else if (data.player.isOp() && ((_b = (_a = data.itemStack) === null || _a === void 0 ? void 0 : _a.nameTag) === null || _b === void 0 ? void 0 : _b.toLowerCase()) === "create crates" || data.player.hasTag("navperm:admin") && ((_d = (_c = data.itemStack) === null || _c === void 0 ? void 0 : _c.nameTag) === null || _d === void 0 ? void 0 : _d.toLowerCase()) === "create crates") {
            data.cancel = true;
            system.run(() => {
                CratesUI.createCrate(data.player, pos, dimensionId);
            });
        }
    }
    if (data.block.typeId === "minecraft:chest") {
        const pos = {
            x: Math.floor(data.block.location.x),
            y: Math.floor(data.block.location.y),
            z: Math.floor(data.block.location.z),
        };
        const dimensionId = data.block.dimension.id;
        const shop = ChestShop.getShop(pos, dimensionId);
        if (shop && !shop.isOwner(data.player)) {
            data.cancel = true;
            if (data.player.isSneaking)
                return;
            system.run(() => {
                ChestShopUI.buy(data.player, shop.chestShop).then((data) => {
                    shop.playerBuy(data.player, data.amount);
                }).catch((err) => {
                    data.player.sendMessage(err);
                });
            });
            return;
        }
        if (((_e = data.itemStack) === null || _e === void 0 ? void 0 : _e.typeId) === "candra:ticket_shop" && !data.player.isSneaking) {
            data.cancel = true;
            if (data.player.isSneaking && !protect(data.player, data.block.location))
                return;
            system.run(() => {
                ChestShopUI.createShop(data.player).then((v) => {
                    ChestShop.createShop(v.player, v.item, v.amount, v.price, pos, dimensionId);
                });
            });
            return;
        }
        if (shop && shop.isOwner(data.player))
            data.cancel = false;
    }
    if (!protect(data.player, data.block.location) && data.itemStack && ((_f = data.itemStack.nameTag) === null || _f === void 0 ? void 0 : _f.toLowerCase()) === "claim wand") {
        PlayerLandPos.setPos1(data.player, data.block.location, data.block.dimension.id);
        data.player.sendMessage(`[Land] §aSet 1`);
        data.cancel = true;
    }
    if (data.block.typeId === "minecraft:decorated_pot" && ((_g = data.itemStack) === null || _g === void 0 ? void 0 : _g.typeId) === "candra:navigation_menu")
        data.cancel = true;
});
world.beforeEvents.itemUseOn.subscribe((data) => {
    data.cancel = protect(data.source, data.block.location);
    if (data.block.typeId === "minecraft:chest") {
        const pos = {
            x: Math.floor(data.block.location.x),
            y: Math.floor(data.block.location.y),
            z: Math.floor(data.block.location.z),
        };
        const dimensionId = data.block.dimension.id;
        const shop = ChestShop.getShop(pos, dimensionId);
        if (shop && !shop.isOwner(data.source)) {
            data.cancel = true;
            if (data.source.isSneaking)
                return;
            system.run(() => {
                ChestShopUI.buy(data.source, shop.chestShop).then((data) => {
                    shop.playerBuy(data.player, data.amount);
                }).catch((err) => {
                    data.source.sendMessage(err);
                });
            });
            return;
        }
        if (data.itemStack.typeId === "candra:ticket_shop" && !data.source.isSneaking) {
            data.cancel = true;
            if (!protect(data.source, data.block.location))
                return;
            system.run(() => {
                ChestShopUI.createShop(data.source).then((v) => {
                    ChestShop.createShop(v.player, v.item, v.amount, v.price, pos, dimensionId);
                });
            });
            return;
        }
        if (shop && shop.isOwner(data.source))
            data.cancel = false;
    }
});
