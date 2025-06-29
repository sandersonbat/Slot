SlotGame/
├── Assets/
│   └── Symbols/   (com símbolos simples tipo A.png, B.png, etc)
├── Main.tscn
├── Main.gd
├── SaveManager.gd
├── PaylineOverlay.gd
├── Outros scripts necessários
├── project.godot
extends Node
const SAVE_PATH = "user://slot_save.dat"

func save_data(data: Dictionary):
    var file = File.new()
    file.open(SAVE_PATH, File.WRITE)
    file.store_var(data)
    file.close()

func load_data() -> Dictionary:
    var file = File.new()
    if file.file_exists(SAVE_PATH):
        file.open(SAVE_PATH, File.READ)
        var data = file.get_var()
        file.close()
        return data
    return {}
    extends Node2D

var winning_lines = []
var payline_colors = [Color.red, Color.green, Color.blue, Color.yellow, Color.purple]
var line_positions = [
    [Vector2(50, 50), Vector2(150, 50), Vector2(250, 50), Vector2(350, 50), Vector2(450, 50)],
    [Vector2(50, 150), Vector2(150, 150), Vector2(250, 150), Vector2(350, 150), Vector2(450, 150)],
    [Vector2(50, 250), Vector2(150, 250), Vector2(250, 250), Vector2(350, 250), Vector2(450, 250)],
    [Vector2(50, 50), Vector2(150, 150), Vector2(250, 250), Vector2(350, 150), Vector2(450, 50)],
    [Vector2(50, 250), Vector2(150, 150), Vector2(250, 50), Vector2(350, 150), Vector2(450, 250)]
]

func show_winning_lines(lines):
    winning_lines = lines
    update()

func _draw():
    for i in winning_lines:
        var color = payline_colors[i % payline_colors.size()]
        var positions = line_positions[i]
        for j in range(positions.size() - 1):
            draw_line(positions[j], positions[j + 1], color, 3)
            extends Node2D

var symbols = ["A", "B", "C"]
var paylines = [
    [0,0,0,0,0], [1,1,1,1,1], [2,2,2,2,2],
    [0,1,2,1,0], [2,1,0,1,2]
]
var current_symbols = []
var credits = 100
var bet = 10
onready var save_manager = preload("res://Scripts/SaveManager.gd").new()

func _ready():
    randomize()
    var data = save_manager.load_data()
    credits = data.get("credits", credits)
    reset_reels()
    update_ui()

func update_ui():
    $CreditsLabel.text = "Créditos: %d" % credits
    $BetLabel.text = "Aposta: %d" % bet
    $ResultLabel.text = ""
    $PaylineOverlay.show_winning_lines([])

func reset_reels():
    current_symbols.clear()
    for reel in range(5):
        var col = []
        for row in range(3):
            var sym = symbols[randi() % symbols.size()]
            col.append(sym)
            update_reel_display(reel, row, sym)
        current_symbols.append(col)

func update_reel_display(reel_idx, row_idx, sym):
    var node = $"ReelContainer/Reel{reel_idx+1}/Row{row_idx+1}"
    node.texture = load("res://Assets/Symbols/%s.png" % sym)

func on_SpinButton_pressed():
    if credits < bet:
        $ResultLabel.text = "Créditos insuficientes!"
        return
    credits -= bet
    update_ui()
    $SpinSound.play()
    reset_reels()
    var lines_won = check_result()
    if lines_won.size() > 0:
        $WinSound.play()
        $WinParticles.restart()
        highlight_winning_symbols(lines_won)
        $PaylineOverlay.show_winning_lines(lines_won.map(lambda x: x - 1))
    save_manager.save_data({"credits": credits})

func check_result() -> Array:
    var payout = 0
    var lines = []
    for i in range(paylines.size()):
        var line = paylines[i]
        var first = current_symbols[0][line[0]]
        var match = true
        for reel in range(1,5):
            if current_symbols[reel][line[reel]] != first:
                match = false
                break
        if match:
            payout += bet * 5
            lines.append(i + 1)
    credits += payout
    $ResultLabel.text = lines.size() > 0 ? 
        "Linhas %s: +%d créditos" % [str(lines), payout] :
        "Nenhuma linha premiada!"
    return lines

func highlight_winning_symbols(lines):
    for line_idx in lines:
        var line = paylines[line_idx - 1]
        for reel in range(5):
            $"ReelContainer/Reel{reel+1}/Row{line[reel]+1}".modulate = Color(1,0.8,0.5)
            
