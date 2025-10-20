// These are "use" statements - they import code libraries we need
use ::rand::Rng;        // For generating random numbers
use ::rand::thread_rng; // For creating a random number generator
use macroquad::prelude::*; // For graphics, input, and window management

/// CONSTANTS: These are fixed values that never change during the program
/// Think of them like unchangeable settings

/// The width of the display in pixels (64 pixels wide, like an old calculator screen)
const DISPLAY_WIDTH: usize = 64;
/// The height of the display in pixels (32 pixels tall)
const DISPLAY_HEIGHT: usize = 32;
/// The size of ram in bytes (4096 bytes = 4KB, tiny by today's standards!)
const RAM_SIZE: usize = 4096;
/// How many CPU cycles happen per frame (controls emulation speed)
const CYCLES_PER_FRAME: usize = 5;

/// RAM STRUCTURE
/// Think of RAM like a giant bookshelf with 4096 numbered shelves.
/// Each shelf holds one byte (a number from 0-255).
/// The CHIP-8 uses this memory layout:
/// 0x000 to 0x080: Reserved for built-in font data (letters/numbers)
/// 0x200: Where most games start loading
/// 0x600: Where some special programs start
/// 0xfff: The last memory address
#[derive(Debug, Copy, Clone)] // These let us copy and debug this struct
struct RAM {
    bytes: [u8; RAM_SIZE], // An array of 4096 bytes (u8 = unsigned 8-bit number, 0-255)
}

impl RAM {
    /// Creates new RAM with the built-in font already loaded
    fn with_fonts() -> Self {
        // Start with all memory set to 0
        let mut ram = Self {
            bytes: [0; RAM_SIZE],
        };

        // FONTSET: These hex numbers represent pixel patterns for digits 0-F
        // Each character is 5 bytes tall, 8 pixels wide
        // 0xF0 in binary is 11110000 (4 pixels on, 4 off)
        let fontset = vec![
            0xF0, 0x90, 0x90, 0x90, 0xF0, //0 - looks like: ████░░░░
            0x20, 0x60, 0x20, 0x20, 0x70, //1            ░░█░░░░░
            0xF0, 0x10, 0xF0, 0x80, 0xF0, //2            ░██░░░░░
            0xF0, 0x10, 0xF0, 0x10, 0xF0, //3            ░░█░░░░░
            0x90, 0x90, 0xF0, 0x10, 0x10, //4            ░███░░░░
            0xF0, 0x80, 0xF0, 0x10, 0xF0, //5
            0xF0, 0x80, 0xF0, 0x90, 0xF0, //6
            0xF0, 0x10, 0x20, 0x40, 0x40, //7
            0xF0, 0x90, 0xF0, 0x90, 0xF0, //8
            0xF0, 0x90, 0xF0, 0x10, 0xF0, //9
            0xF0, 0x90, 0xF0, 0x90, 0x90, //a
            0xE0, 0x90, 0xE0, 0x90, 0xE0, //b
            0xF0, 0x80, 0x80, 0x80, 0xF0, //c
            0xE0, 0x90, 0x90, 0x90, 0xE0, //d
            0xF0, 0x80, 0xF0, 0x80, 0xF0, //e
            0xF0, 0x80, 0xF0, 0x80, 0x80, //f
        ];

        // Copy the fontset into the beginning of RAM
        // This loop goes through each font byte and puts it in memory
        for (idx, value) in ram.bytes[0..fontset.len()].iter_mut().enumerate() {
            *value = fontset[idx];
        }
        ram
    }

    /// Gets a 16-bit value from RAM (combines two 8-bit bytes)
    /// CHIP-8 uses "big endian" - the bigger byte comes first
    /// Like reading "12" as twelve, not twenty-one
    fn get(self, index: u16) -> u16 {
        // Shift first byte left by 8 positions, then add second byte
        // Example: byte1=0x12, byte2=0x34 becomes 0x1234
        ((self.bytes[index as usize] as u16) << 8) | self.bytes[(index + 1) as usize] as u16
    }
}

/// ROM BUFFER: Holds the game file data
/// Think of this like loading a cartridge - it reads the game file into memory
struct RomBuffer {
    buffer: Vec<u8>, // A dynamic list of bytes (the game data)
}

impl RomBuffer {
    /// Creates a new ROM buffer by reading a file
    fn new(file: &str) -> Self {
        // Read the entire file into a vector of bytes
        // .unwrap() means "crash if this fails" (not great practice, but simple)
        let buffer: Vec<u8> = std::fs::read(file).unwrap();
        RomBuffer { buffer: buffer }
    }
}

/// REGISTERS: Think of these as the CPU's personal scratch paper
/// A register is like a small box that holds one number the CPU is working with
/// The CHIP-8 has 16 general-purpose registers (V0 through VF)
/// Plus some special-purpose registers for specific tasks
#[derive(Clone, Copy)]
struct Registers {
    register: [u8; 16],  // 16 general-purpose registers (V0-VF)
    vindex: u16,         // Index register (I) - points to memory locations
    delay_timer: u8,     // Counts down at 60Hz, used for timing
    sound_timer: u8,     // When >0, makes a beep sound
}

impl Registers {
    /// Creates new registers with all values set to 0
    fn new() -> Self {
        Registers {
            register: [0u8; 16], // All 16 registers start at 0
            vindex: 0,           // Index register starts at 0
            delay_timer: 0,      // Timer starts at 0
            sound_timer: 0,      // Timer starts at 0
        }
    }
    
    /// Sets the index register (I) to a specific value
    /// The index register points to memory addresses
    fn set_index_register(&mut self, value: u16) {
        self.vindex = value;
    }
    
    /// Gets the current value of the index register
    fn get_index_register(&self) -> u16 {
        self.vindex
    }
    
    /// Sets the sound timer (makes beeping when > 0)
    fn set_sound_timer(&mut self, value: u8) {
        self.sound_timer = value;
    }
    
    /// Sets the delay timer (used for game timing)
    fn set_delay_timer(&mut self, value: u8) {
        self.delay_timer = value;
    }
    
    /// Gets the current delay timer value
    fn get_delay_timer(&self) -> u8 {
        return self.delay_timer;
    }
    
    /// Decreases sound timer by 1 (but never goes below 0)
    fn decrement_sound_timer(&mut self) {
        if self.sound_timer > 0 {
            self.sound_timer -= 1;
        }
    }
    
    /// Decreases delay timer by 1 (but never goes below 0)
    fn decrement_delay_timer(&mut self) {
        if self.delay_timer > 0 {
            self.delay_timer -= 1;
        }
    }

    /// Gets the value stored in a specific register (0-15)
    fn get_register(&self, register: u8) -> u8 {
        return self.register[register as usize];
    }
    
    /// Sets a specific register to a value
    fn set_register(&mut self, register: u8, value: u8) {
        self.register[register as usize] = value;
    }
}

/// STACK: Like a stack of plates - you can only add/remove from the top
/// Used when the CPU calls subroutines (like functions) and needs to remember where to return
/// Can handle 16 levels of nested function calls
#[derive(Clone, Copy)]
struct Stack {
    values: [u16; 16], // 16 memory addresses (where to return to)
}

impl Stack {
    /// Creates a new empty stack
    fn new() -> Self {
        Stack { values: [0; 16] }
    }
}

/// CPU: The main processor that runs everything
/// Think of this as the brain of the CHIP-8 computer
struct CPU {
    display: [[bool; DISPLAY_WIDTH]; DISPLAY_HEIGHT], // 64x32 grid of pixels (on/off)
    program_counter: u16,    // Points to the current instruction in memory
    memory: RAM,             // The 4KB of RAM
    registers: Registers,    // All the CPU registers
    stack: Stack,           // The call stack
    stackpointer: u8,       // Points to the top of the stack (0-15)
}

impl CPU {
    /// FETCH: Gets the next instruction from memory
    /// Like reading the next line in a recipe
    fn fetch(&self, ram: &RAM) -> u16 {
        // Get 2 bytes from memory at the program counter location
        ram.get(self.program_counter)
    }
    
    /// DECODE: Figures out what the instruction means
    /// Like understanding "add 2 cups flour" from a recipe
    fn decode(&self, opcode: u16) -> Instruction {
        // Instructions are identified by their first few bits
        // This massive match statement handles all possible instructions
        match self.first_nibble(opcode) {
            0x0 => match self.last_byte(opcode) {
                0xE0 => Instruction::ClearScreen,           // Clear the display
                0xEE => Instruction::ReturnFromSubroutine,  // Return from function call
                _ => Instruction::NOOP,                     // Do nothing (unknown instruction)
            },
            0x1 => Instruction::JUMP {                      // Jump to address
                nnn: self.oxxx(opcode),
            },
            0x2 => Instruction::CallSubroutineAtNNN {       // Call function
                nnn: self.oxxx(opcode),
            },
            0x3 => Instruction::SkipNextInstructionIfXIsKK { // Skip if register equals value
                x: self.second_nibble(opcode),
                kk: self.last_byte(opcode),
            },
            0x4 => Instruction::SkipNextInstructionIfXIsNotKK { // Skip if register not equals value
                x: self.second_nibble(opcode),
                kk: self.last_byte(opcode),
            },
            0x5 => Instruction::SkipNextInstructionIfXIsY { // Skip if two registers equal
                x: self.second_nibble(opcode),
                y: self.third_nibble(opcode),
            },
            0x6 => Instruction::LoadRegisterX {             // Set register to value
                x: self.second_nibble(opcode),
                kk: self.last_byte(opcode),
            },
            0x7 => Instruction::AddToRegisterX {            // Add value to register
                x: self.second_nibble(opcode),
                kk: self.last_byte(opcode),
            },
            0x8 => match self.fourth_nibble(opcode) {       // Math and logic operations
                0x0 => Instruction::LoadRegisterXIntoY {    // Copy register Y into X
                    x: self.second_nibble(opcode),
                    y: self.third_nibble(opcode),
                },
                0x1 => Instruction::LoadXOrYinX {           // X = X OR Y
                    x: self.second_nibble(opcode),
                    y: self.third_nibble(opcode),
                },
                0x2 => Instruction::LoadXAndYInX {          // X = X AND Y
                    x: self.second_nibble(opcode),
                    y: self.third_nibble(opcode),
                },
                0x3 => Instruction::LoadXXorYInX {          // X = X XOR Y
                    x: self.second_nibble(opcode),
                    y: self.third_nibble(opcode),
                },
                0x4 => Instruction::AddYToX {               // X = X + Y
                    x: self.second_nibble(opcode),
                    y: self.third_nibble(opcode),
                },
                0x5 => Instruction::SubYFromX {             // X = X - Y
                    x: self.second_nibble(opcode),
                    y: self.third_nibble(opcode),
                },
                0x6 => Instruction::ShiftXRight1 {          // X = X >> 1 (divide by 2)
                    x: self.second_nibble(opcode),
                },
                0x7 => Instruction::SubXFromY {             // X = Y - X
                    x: self.second_nibble(opcode),
                    y: self.third_nibble(opcode),
                },
                0xE => Instruction::ShiftXLeft1 {           // X = X << 1 (multiply by 2)
                    x: self.second_nibble(opcode),
                },
                _ => {
                    panic!("some other 8xxx thingy")        // Crash if unknown instruction
                }
            },
            0x9 => Instruction::SkipNextInstructionIfXIsNotY { // Skip if registers not equal
                x: self.second_nibble(opcode),
                y: self.third_nibble(opcode),
            },
            0xA => Instruction::SetIndexRegister {          // Set index register I
                nnn: self.oxxx(opcode),
            },
            0xB => Instruction::JumpToAddressPlusV0 {       // Jump to address + V0
                nnn: self.oxxx(opcode),
            },
            0xC => Instruction::SetXToRandom {              // X = random number AND value
                x: self.second_nibble(opcode),
                kk: self.last_byte(opcode),
            },
            0xD => Instruction::DISPLAY {                   // Draw sprite on screen
                x: self.second_nibble(opcode),
                y: self.third_nibble(opcode),
                n: self.fourth_nibble(opcode),
            },
            0xE => match self.last_byte(opcode) {           // Key input checks
                0xA1 => Instruction::SkipIfVxNotPressed {   // Skip if key not pressed
                    x: self.second_nibble(opcode),
                },
                0x9E => Instruction::SkipIfVxPressed {      // Skip if key pressed
                    x: self.second_nibble(opcode),
                },
                _ => {
                    panic!("unimplemented opcode: 0x{:04x}", opcode);
                }
            },
            0xF => match self.last_byte(opcode) {           // Misc operations
                0x0A => Instruction::WaitForKeyPressed {    // Wait for key press
                    x: self.second_nibble(opcode),
                },
                0x07 => Instruction::SetXToDelayTimer {     // X = delay timer
                    x: self.second_nibble(opcode),
                },
                0x15 => Instruction::SetDelayTimerToX {     // delay timer = X
                    x: self.second_nibble(opcode),
                },
                0x18 => Instruction::SetSoundTimerToX {     // sound timer = X
                    x: self.second_nibble(opcode),
                },
                0x1E => Instruction::AddXtoI {              // I = I + X
                    x: self.second_nibble(opcode),
                },
                0x29 => Instruction::SetIToSpriteX {        // I = location of digit X
                    x: self.second_nibble(opcode),
                },
                0x33 => Instruction::LoadBCDOfX {           // Store BCD of X in memory
                    x: self.second_nibble(opcode),
                },
                0x55 => Instruction::Write0ThroughX {       // Store V0-VX in memory
                    x: self.second_nibble(opcode),
                },
                0x65 => Instruction::Load0ThroughX {        // Load V0-VX from memory
                    x: self.second_nibble(opcode),
                },
                _ => {
                    panic!("unimplemented opcode: 0x{:06x}", opcode);
                }
            },
            _ => {
                panic!("cannot decode,opcode not implemented: 0x{:04x}", opcode)
            }
        }
    }
    
    /// Converts a CHIP-8 key code (0-F) to a keyboard key
    /// CHIP-8 had a 16-key hexadecimal keypad, we map it to modern keyboard
    fn u8_to_keycode(code: u8) -> macroquad::input::KeyCode {
        match code {
            0x0 => macroquad::input::KeyCode::X,      // 0 -> X key
            0x1 => macroquad::input::KeyCode::Key1,   // 1 -> 1 key
            0x2 => macroquad::input::KeyCode::Key2,   // 2 -> 2 key
            0x3 => macroquad::input::KeyCode::Key3,   // 3 -> 3 key
            0x4 => macroquad::input::KeyCode::Q,      // 4 -> Q key
            0x5 => macroquad::input::KeyCode::W,      // 5 -> W key
            0x6 => macroquad::input::KeyCode::E,      // 6 -> E key
            0x7 => macroquad::input::KeyCode::A,      // 7 -> A key
            0x8 => macroquad::input::KeyCode::S,      // 8 -> S key
            0x9 => macroquad::input::KeyCode::D,      // 9 -> D key
            0xa => macroquad::input::KeyCode::Z,      // A -> Z key
            0xb => macroquad::input::KeyCode::C,      // B -> C key
            0xc => macroquad::input::KeyCode::Key3,   // C -> 3 key (duplicate?)
            0xd => macroquad::input::KeyCode::R,      // D -> R key
            0xe => macroquad::input::KeyCode::F,      // E -> F key
            0xf => macroquad::input::KeyCode::V,      // F -> V key
            _ => {
                panic!("Invalid keycode");            // Crash if invalid key
            }
        }
    }

    /// EXECUTE: Actually performs the instruction
    /// Like following the recipe step: "add 2 cups flour"
    fn execute(&mut self, instruction: Instruction) {
        match instruction {
            Instruction::NOOP => {
                // Do absolutely nothing
            }
            
            // 00E0 - Clear the screen (set all pixels to off/false)
            Instruction::ClearScreen => {
                self.display
                    .iter_mut()                              // Go through each row
                    .for_each(|x| *x = [false; DISPLAY_WIDTH]); // Set all pixels in row to false
            }
            
            // 00EE - Return from subroutine (like returning from a function call)
            Instruction::ReturnFromSubroutine => {
                self.stackpointer -= 1;                      // Move stack pointer down
                self.program_counter = self.stack.values[self.stackpointer as usize]; // Jump back
            }
            
            // 1NNN - Jump to address NNN
            Instruction::JUMP { nnn } => {
                println!("Jumping to {:?}", nnn);           // Debug output
                self.program_counter = nnn;                 // Set program counter to new address
            }
            
            // 2NNN - Call subroutine at NNN (like calling a function)
            Instruction::CallSubroutineAtNNN { nnn } => {
                self.stack.values[self.stackpointer as usize] = self.program_counter; // Save current location
                self.stackpointer += 1;                     // Move stack pointer up
                self.program_counter = nnn;                 // Jump to subroutine
            }
            
            // 3XKK - Skip next instruction if VX equals KK
            Instruction::SkipNextInstructionIfXIsKK { x, kk } => {
                let vx = self.registers.get_register(x);   // Get register X value
                if vx == kk {                               // If it equals KK
                    self.program_counter += 2;              // Skip next instruction (add 2 bytes)
                }
            }
            
            // 4XKK - Skip next instruction if VX does NOT equal KK
            Instruction::SkipNextInstructionIfXIsNotKK { x, kk } => {
                let vx = self.registers.get_register(x);   // Get register X value
                if vx != kk {                               // If it doesn't equal KK
                    self.program_counter += 2;              // Skip next instruction
                }
            }
            
            // 5XY0 - Skip next instruction if VX equals VY
            Instruction::SkipNextInstructionIfXIsY { x, y } => {
                let vx = self.registers.get_register(x);   // Get register X value
                let vy = self.registers.get_register(y);   // Get register Y value
                if vx == vy {                               // If they're equal
                    self.program_counter += 2;              // Skip next instruction
                }
            }
            
            // 6XKK - Load value KK into register VX
            Instruction::LoadRegisterX { x, kk } => {
                self.registers.set_register(x, kk);        // VX = KK
            }
            
            // 7XKK - Add KK to register VX
            Instruction::AddToRegisterX { x, kk } => {
                let vx = self.registers.get_register(x);   // Get current VX value
                let (tmp, _overflow) = vx.overflowing_add(kk); // Add KK (handle overflow)
                self.registers.set_register(x, tmp);       // Store result back in VX
            }
            
            // 8XY0 - Load VY into VX
            Instruction::LoadRegisterXIntoY { x, y } => {
                let vy = self.registers.get_register(y);   // Get VY value
                self.registers.set_register(x, vy);        // VX = VY
            }
            
            // 8XY1 - VX = VX OR VY (bitwise OR operation)
            Instruction::LoadXOrYinX { x, y } => {
                let vx = self.registers.get_register(x);   // Get VX
                let vy = self.registers.get_register(y);   // Get VY
                self.registers.set_register(x, vx | vy);   // VX = VX OR VY
            }
            
            // 8XY2 - VX = VX AND VY (bitwise AND operation)
            Instruction::LoadXAndYInX { x, y } => {
                let vx = self.registers.get_register(x);   // Get VX
                let vy = self.registers.get_register(y);   // Get VY
                self.registers.set_register(x, vx & vy);   // VX = VX AND VY
            }
            
            // 8XY3 - VX = VX XOR VY (bitwise exclusive OR)
            Instruction::LoadXXorYInX { x, y } => {
                let vx = self.registers.get_register(x);   // Get VX
                let vy = self.registers.get_register(y);   // Get VY
                self.registers.set_register(x, vx ^ vy);   // VX = VX XOR VY
            }
            
            // 8XY4 - VX = VX + VY (with carry flag in VF)
            Instruction::AddYToX { x, y } => {
                let vx = self.registers.get_register(x);   // Get VX
                let vy = self.registers.get_register(y);   // Get VY
                let (res, fv) = vy.overflowing_add(vx);    // Add and check for overflow
                self.registers.set_register(x, res);       // Store result
                self.registers.set_register(0xf, if fv { 1 } else { 0 }); // Set carry flag
            }

            // 8XY5 - VX = VX - VY (with borrow flag in VF)
            Instruction::SubYFromX { x, y } => {
                let vx = self.registers.get_register(x);   // Get VX
                let vy = self.registers.get_register(y);   // Get VY
                let (res, fv) = vx.overflowing_sub(vy);    // Subtract and check for underflow
                self.registers.set_register(x, res);       // Store result
                self.registers.set_register(0xf, if fv { 0 } else { 1 }); // Set borrow flag
            }

            // 8XY6 - Shift VX right by 1 bit (divide by 2)
            Instruction::ShiftXRight1 { x } => {
                let vx = self.registers.get_register(x);   // Get VX
                let vf = if vx & 1 == 1 { 1 } else { 0 };  // Check if lowest bit is 1
                self.registers.set_register(x, vx.overflowing_shr(1).0); // Shift right
                self.registers.set_register(0xF, vf);      // Store shifted bit in VF
            }

            // 8XYE - Shift VX left by 1 bit (multiply by 2)
            Instruction::ShiftXLeft1 { x } => {
                let vx = self.registers.get_register(x);   // Get VX
                let fv = (vx as u16 >> 7) & 1;             // Check highest bit
                let res = self.registers.get_register(x).wrapping_shl(1); // Shift left
                self.registers.set_register(x, res);       // Store result
                self.registers.set_register(0xf, if fv == 1 { 1 } else { 0 }); // Store shifted bit
            }
            
            // 8XY7 - VX = VY - VX (reverse subtraction)
            Instruction::SubXFromY { x, y } => {
                let vx = self.registers.get_register(x);   // Get VX
                let vy = self.registers.get_register(y);   // Get VY
                let (res, fv) = vy.overflowing_sub(vx);    // VY - VX
                self.registers.set_register(x, res);       // Store in VX
                self.registers.set_register(0xf, if fv { 0 } else { 1 }); // Set borrow flag
            }

            // 9XY0 - Skip next instruction if VX != VY
            Instruction::SkipNextInstructionIfXIsNotY { x, y } => {
                let vx = self.registers.get_register(x);   // Get VX
                let vy = self.registers.get_register(y);   // Get VY
                if vx != vy {                               // If they're different
                    self.program_counter += 2;              // Skip next instruction
                }
            }
            
            // ANNN - Set index register I to NNN
            Instruction::SetIndexRegister { nnn } => {
                self.registers.set_index_register(nnn);    // I = NNN
            }
            
            // BNNN - Jump to address NNN + V0
            Instruction::JumpToAddressPlusV0 { nnn } => {
                let v0 = (self.registers.get_register(0) & 0xf) as u16; // Get V0 (only low 4 bits)
                self.program_counter = nnn + v0;            // Jump to NNN + V0
            }
            
            // CXKK - Set VX to random number AND KK
            Instruction::SetXToRandom { x, kk } => {
                let mut rng = thread_rng();                 // Create random number generator
                let random_number = rng.gen_range(0..=255); // Generate random 0-255
                self.registers.set_register(x, random_number & kk); // VX = random AND KK
            }
            
            // DXYN - Draw sprite at (VX, VY) with height N
            // This is the most complex instruction - it draws graphics!
            Instruction::DISPLAY { x, y, n } => {
                // Get starting coordinates from registers, wrap around screen edges
                let start_x = (self.registers.get_register(x) % DISPLAY_WIDTH as u8) as usize;
                let start_y = (self.registers.get_register(y) % DISPLAY_HEIGHT as u8) as usize;

                // Get sprite data location from index register
                let sprite_start = self.registers.get_index_register() as usize;
                self.registers.set_register(0xF, 0);       // Clear collision flag

                // Draw each row of the sprite (N rows total)
                for sprite_row in 0..n as usize {
                    if sprite_start + sprite_row >= RAM_SIZE {
                        return;                              // Don't read past end of memory
                    }
                    let sprite = self.memory.bytes[sprite_start + sprite_row]; // Get sprite row data
                    
                    // Draw each pixel in the row (8 pixels wide)
                    for sprite_column in 0..8 as usize {
                        let pixel_row = start_x + sprite_column;    // Screen X coordinate
                        let pixel_column = start_y + sprite_row;    // Screen Y coordinate

                        // Check if this sprite pixel should be drawn (bit is 1)
                        let sprite_pixel_set = sprite >> (7 - sprite_column) & 1 == 1;

                        // Only draw if we're within screen bounds
                        if pixel_row < DISPLAY_WIDTH && pixel_column < DISPLAY_HEIGHT {
                            // Check for collision (both sprite and screen pixel are on)
                            if self.display[pixel_column][pixel_row] && sprite_pixel_set {
                                self.registers.set_register(0xf, 1); // Set collision flag
                            }
                            // XOR the sprite pixel with screen pixel (CHIP-8 drawing rule)
                            self.display[pixel_column][pixel_row] ^= sprite_pixel_set;
                        }
                    }
                }
            }
            
            // EXA1 - Skip next instruction if key VX is NOT pressed
            Instruction::SkipIfVxNotPressed { x } => {
                let key = CPU::u8_to_keycode(self.registers.get_register(x) & 0xf); // Get key
                if !is_key_down(key) {                      // If key is not pressed
                    self.program_counter += 2;              // Skip next instruction
                }
            }
            
            // EX9E - Skip next instruction if key VX IS pressed
            Instruction::SkipIfVxPressed { x } => {
                let key = CPU::u8_to_keycode(self.registers.get_register(x) & 0xf); // Get key
                if is_key_down(key) {                       // If key is pressed
                    self.program_counter += 2;              // Skip next instruction
                }
            }
            
            // FX0A - Wait for key press (halt until key pressed)
            Instruction::WaitForKeyPressed { x } => {
                // TODO: This instruction should halt the CPU until a key is pressed
                // Currently does nothing - this is a bug!
                let _key = CPU::u8_to_keycode(self.registers.get_register(x) & 0xf);
            }
            
            // FX07 - Set VX to current delay timer value
            Instruction::SetXToDelayTimer { x } => {
                let vdt = self.registers.get_delay_timer();
                self.registers.set_register(x, vdt);       // VX = delay timer
            }
            
            // FX15 - Set delay timer to VX
            Instruction::SetDelayTimerToX { x } => {
                let vx = self.registers.get_register(x);
                self.registers.set_delay_timer(vx);        // delay timer = VX
            }
            
            // FX18 - Set sound timer to VX
            Instruction::SetSoundTimerToX { x } => {
                let vx = self.registers.get_register(x);
                self.registers.set_sound_timer(vx);        // sound timer = VX
            }
            
            // FX1E - Add VX to index register I
            Instruction::AddXtoI { x } => {
                let vx = self.registers.get_register(x) as u16; // Get VX as 16-bit
                let vi = self.registers.get_index_register();   // Get current I
                let added = vi + vx;                             // Add them
                self.registers.set_index_register(added);       // I = I + VX
            }
            
            // FX29 - Set I to location of sprite for digit VX
            Instruction::SetIToSpriteX { x } => {
                let vx = (self.registers.get_register(x) * 5) as u16 & 0xffff; // Each font is 5 bytes
                self.registers.set_index_register(vx);          // I = location of digit
            }
            
            // FX33 - Store BCD (Binary Coded Decimal) of VX in memory
            // Converts number to 3 decimal digits: hundreds, tens, ones
            Instruction::LoadBCDOfX { x } => {
                let vx = self.registers.get_register(x);       // Get VX value
                let store_index = self.registers.get_index_register() as usize; // Where to store
                self.memory.bytes[store_index] = vx / 100;      // Hundreds digit
                self.memory.bytes[store_index + 1] = (vx % 100) / 10; // Tens digit
                self.memory.bytes[store_index + 2] = (vx % 100) % 10;  // Ones digit
            }
            
            // FX55 - Store registers V0 through VX in memory starting at I
            Instruction::Write0ThroughX { x } => {
                let vi = self.registers.get_index_register() as usize; // Memory start location
                for register in 0..x + 1 {                     // For each register 0 to X
                    let register_value = self.registers.get_register(register); // Get register value
                    self.memory.bytes[vi + register as usize] = register_value; // Store in memory
                }
            }
            
            // FX65 - Load registers V0 through VX from memory starting at I
            Instruction::Load0ThroughX { x } => {
                let vi = self.registers.get_index_register() as usize; // Memory start location
                for i in 0..x + 1 {                             // For each register 0 to X
                    self.registers.set_register(i, self.memory.bytes[vi + i as usize]); // Load from memory
                }
            }
        }
    }
    
    /// HELPER FUNCTIONS for breaking apart instruction opcodes
    /// An instruction is 16 bits (2 bytes) that needs to be broken into pieces
    
    /// Gets the first 4 bits (nibble) of an opcode
    /// Example: 0x1234 -> returns 0x1
    fn first_nibble(&self, opcode: u16) -> u8 {
        ((opcode >> 12) & 0xF) as u8
    }
    
    /// Gets the second 4 bits of an opcode
    /// Example: 0x1234 -> returns 0x2  
    fn second_nibble(&self, opcode: u16) -> u8 {
        ((opcode >> 8) & 0xf) as u8
    }
    
    /// Gets the third 4 bits of an opcode
    /// Example: 0x1234 -> returns 0x3
    fn third_nibble(&self, opcode: u16) -> u8 {
        ((opcode >> 4) & 0xf) as u8
    }
    
    /// Gets the fourth 4 bits of an opcode
    /// Example: 0x1234 -> returns 0x4
    fn fourth_nibble(&self, opcode: u16) -> u8 {
        (opcode as u8) & 0xf
    }
    
    /// Gets the last 8 bits (1 byte) of an opcode
    /// Example: 0x1234 -> returns 0x34
    fn last_byte(&self, opcode: u16) -> u8 {
        (opcode & 0xff) as u8
    }
    
    /// Gets the last 12 bits of an opcode (used for addresses)
    /// Example: 0x1234 -> returns 0x234
    fn oxxx(&self, opcode: u16) -> u16 {
        opcode & 0xfff
    }

    /// The main CPU cycle: Fetch -> Decode -> Execute
    /// This is the heart of the emulator - called repeatedly to run the program
    fn cycle(&mut self) {
        let opcode = self.fetch(&self.memory);          // Fetch next instruction
        self.program_counter += 2;                      // Move to next instruction (2 bytes)
        let instruction = self.decode(opcode);          // Decode what to do
        self.execute(instruction);                      // Execute the instruction
        
        // Update timers (these should count down at 60Hz)
        self.registers.decrement_sound_timer();        // Countdown sound timer
        self.registers.decrement_delay_timer();        // Countdown delay timer
    }

    /// Creates a new CPU with a ROM loaded into memory
    fn new(rom: RomBuffer) -> Self {
        let mut memory = RAM::with_fonts();             // Start with fonts loaded
        
        // Load the ROM data into memory starting at address 0x200
        // This is where CHIP-8 programs traditionally start
        for (x, y) in rom.buffer.iter().enumerate() {
            memory.bytes[0x200 + x] = *y;               // Copy each ROM byte to memory
        }

        Self {
            display: [[false; DISPLAY_WIDTH]; DISPLAY_HEIGHT], // All pixels start off
            program_counter: 0x200,                     // Start execution at 0x200
            registers: Registers::new(),                // All registers start at 0
            memory: memory,                             // Memory with ROM loaded
            stack: Stack::new(),                        // Empty call stack
            stackpointer: 0,                            // Stack pointer at bottom
        }
    }
}

/// INSTRUCTION ENUM: Defines every possible CHIP-8 instruction
/// Think of this as a list of every possible "command" the CPU can understand
/// Each instruction has a name and may include data (like addresses or values)
enum Instruction {
    NOOP,                                               // Do nothing
    ClearScreen,                                        // Clear display
    ReturnFromSubroutine,                              // Return from function
    JUMP { nnn: u16 },                                 // Jump to address
    CallSubroutineAtNNN { nnn: u16 },                  // Call function
    LoadRegisterX { x: u8, kk: u8 },                  // Set register
    AddToRegisterX { x: u8, kk: u8 },                 // Add to register
    LoadXOrYinX { x: u8, y: u8 },                     // Binary OR
    LoadXAndYInX { x: u8, y: u8 },                    // Binary AND
    LoadXXorYInX { x: u8, y: u8 },                    // Binary XOR
    AddYToX { x: u8, y: u8 },                         // Add registers
    SubYFromX { x: u8, y: u8 },                       // Subtract registers
    ShiftXRight1 { x: u8 },                           // Bit shift right
    ShiftXLeft1 { x: u8 },                            // Bit shift left
    SubXFromY { x: u8, y: u8 },                       // Reverse subtract
    LoadRegisterXIntoY { x: u8, y: u8 },              // Copy register
    SetIndexRegister { nnn: u16 },                    // Set I register
    JumpToAddressPlusV0 { nnn: u16 },                 // Jump with offset
    SkipNextInstructionIfXIsKK { x: u8, kk: u8 },     // Conditional skip
    SkipNextInstructionIfXIsNotKK { x: u8, kk: u8 },  // Conditional skip
    SkipNextInstructionIfXIsY { x: u8, y: u8 },       // Conditional skip
    SkipNextInstructionIfXIsNotY { x: u8, y: u8 },    // Conditional skip
    SetXToRandom { x: u8, kk: u8 },                   // Random number
    DISPLAY { x: u8, y: u8, n: u8 },                  // Draw sprite
    SkipIfVxNotPressed { x: u8 },                     // Key input
    SkipIfVxPressed { x: u8 },                        // Key input
    WaitForKeyPressed { x: u8 },                      // Wait for key
    SetXToDelayTimer { x: u8 },                       // Timer operations
    SetDelayTimerToX { x: u8 },                       // Timer operations
    SetSoundTimerToX { x: u8 },                       // Sound timer
    AddXtoI { x: u8 },                                // Add to I register
    SetIToSpriteX { x: u8 },                          // Font location
    LoadBCDOfX { x: u8 },                             // BCD conversion
    Write0ThroughX { x: u8 },                         // Memory dump
    Load0ThroughX { x: u8 },                          // Memory load
}

/// MAIN FUNCTION: This is where the program starts
/// The #[macroquad::main] attribute creates a window and game loop
#[macroquad::main("Chip 8 interpreter \"Chippie\" ")]
async fn main() {
    // Load the CHIP-8 ROM file (in this case, "pong.ch8")
    let b = RomBuffer::new("./pong.ch8");
    let mut c = CPU::new(b);                            // Create CPU with ROM loaded

    // Graphics setup for displaying the CHIP-8 screen
    // Create a 64x32 image (same size as CHIP-8 display)
    let mut image = Image::gen_image_color(DISPLAY_WIDTH as u16, DISPLAY_HEIGHT as u16, WHITE);
    let mut buffer = vec![false; DISPLAY_WIDTH * DISPLAY_HEIGHT]; // Pixel buffer
    let texture = Texture2D::from_image(&image);        // GPU texture
    texture.set_filter(FilterMode::Nearest);            // Pixelated look (no smoothing)

    let mut running = true;                             // Main loop control

    // MAIN GAME LOOP: Runs until user quits
    while running {
        // Check if user pressed Escape key to quit
        if is_key_pressed(KeyCode::Escape) {
            running = false;
            continue;
        }

        // Run several CPU cycles per frame for proper speed
        // CHIP-8 originally ran at ~500Hz, we run 5 cycles per 60Hz frame
        for _ in 0..=CYCLES_PER_FRAME {
            c.cycle();                                  // Execute one CPU instruction
        }

        // GRAPHICS RENDERING
        clear_background(WHITE);                        // Clear screen to white
        
        // Copy CHIP-8 display to our buffer
        // Transform 2D array to 1D array for easier processing
        for y in 0..DISPLAY_HEIGHT as i32 {
            for x in 0..DISPLAY_WIDTH as i32 {
                buffer[y as usize * DISPLAY_WIDTH + x as usize] = c.display[y as usize][x as usize];
            }
        }

        // Convert buffer to image pixels
        for i in 0..buffer.len() {
            image.set_pixel(
                (i % DISPLAY_WIDTH) as u32,             // X coordinate
                (i / DISPLAY_WIDTH) as u32,             // Y coordinate
                match buffer[i as usize] {              // Color based on pixel state
                    true => BLACK,                      // On pixels are black
                    false => WHITE,                     // Off pixels are white
                },
            );
        }
        
        // Update the GPU texture with new image data
        texture.update(&image);

        // Draw the texture scaled to fill the window
        draw_texture_ex(
            &texture,
            0.0,                                        // X position (top-left)
            0.0,                                        // Y position (top-left)
            WHITE,                                      // Tint color (no tint)
            DrawTextureParams {
                dest_size: Some(vec2(screen_width(), screen_height())), // Scale to window size
                ..Default::default()
            },
        );
        
        // Wait for next frame (maintains 60 FPS)
        next_frame().await;
    }
}
