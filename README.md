"""
Alina Hassan
Final Project
"""
import os

gameprogramming = os.path.dirname(__file__)
finalproject = os.path.join(gameprogramming, "gpfinal", "final project.tmx")
playerfile = os.path.join(gameprogramming, "assets", "kenney_pixel-platformer", "Tiles", "Characters", "tile_0001.png")
background = os.path.join(gameprogramming, "gpfinal", "menubackground", "cloudsbackground.jpeg")
fonts = os.path.join(gameprogramming, "assets", "kenney_kenney-fonts", "Fonts", "KenneyBlocks.ttf")

import arcade
from arcade.types import Color

# --- Constants
WINDOW_TITLE = "Cloud Chaser"
WINDOW_WIDTH = 1000
WINDOW_HEIGHT = 700

# Constants used to scale our sprites from their original size
CHARACTER_SCALING = 1.3
TILE_SCALING = 1.9
COIN_SCALING = 1.0
SPRITE_PIXEL_SIZE = 128
GRID_PIXEL_SIZE = SPRITE_PIXEL_SIZE * TILE_SCALING

# Movement speed of player, in pixels per frame
PLAYER_MOVEMENT_SPEED = 5
GRAVITY = 1
PLAYER_JUMP_SPEED = 15

# Camera constants
FOLLOW_DECAY_CONST = 0.3  # get within 1% of the target position within 2 seconds

#create menu screen
#https://www.youtube.com/watch?v=G8MYGDf_9ho
class MenuView(arcade.View):
    def __init__(self):
        super().__init__()
        #add background image
        menu_background = os.path.join(gameprogramming, "gpfinal", "menubackground", "cloud.png")
        self.menu = arcade.load_texture(menu_background) 

        self.title_text = arcade.Text(
            "Cloud Chaser",
            x=WINDOW_WIDTH // 2,
            y=WINDOW_HEIGHT // 1.5 + 50,
            color=arcade.color.DARK_CERULEAN,
            font_name= "Courier New",
            font_size=50,
            anchor_x="center"
        )
        self.rules_text = arcade.Text (
            "Rules:",
            x=WINDOW_WIDTH // 2,
            y=WINDOW_HEIGHT // 1.5 - 50,
            color=arcade.color.DARK_SLATE_BLUE,
            font_name= "Courier New",
            font_size=20,
            anchor_x="center"
        )
        self.rules2_text = arcade.Text (
            "1. Collect all the coins!",
            x=WINDOW_WIDTH // 2,
            y=WINDOW_HEIGHT // 1.5 - 100,
            color=arcade.color.DARK_SLATE_BLUE,
            font_name= "Courier New",
            font_size=20,
            anchor_x="center"
        )
        self.rules3_text = arcade.Text (
            "2. Water is deadly, avoid it!",
            x=WINDOW_WIDTH // 2,
            y=WINDOW_HEIGHT // 1.5 - 150,
            color=arcade.color.DARK_SLATE_BLUE,
            font_name= "Courier New",
            font_size=20,
            anchor_x="center"
        )

        self.instruction_text = arcade.Text(
            "Press ENTER to start",
            x=WINDOW_WIDTH // 2,
            y=WINDOW_HEIGHT // 1.5 - 250,
            color=arcade.color.DARK_CERULEAN,
            font_name= "Courier New",
            font_size=30,
            anchor_x="center"
        )
        #self.background = None
        #self.background = arcade.load_texture(background)
        #arcade.set_background_color(arcade.color.COLUMBIA_BLUE)

    def on_draw(self):
        self.clear()
        arcade.draw_texture_rect(self.menu, 
                                 arcade.LBWH(0, 0, self.width, self.height) 
                                )

        self.title_text.draw()
        self.rules_text.draw()
        self.rules2_text.draw()
        self.rules3_text.draw()
        self.instruction_text.draw()
    def on_key_press(self, key, modifiers):
        if key == arcade.key.ENTER:
            game = GameView()
            game.reset()
            self.window.show_view(game)

class GameView(arcade.View):
    """
    Main application class.
    """
    def __init__(self):
        super().__init__()

        # A Camera that can be used for scrolling the screen
        self.camera_sprites = arcade.Camera2D()

        # A rectangle that is used to constrain the camera's position.
        # we update it when we load the tilemap
        self.camera_bounds = self.window.rect

        # A non-scrolling camera that can be used to draw GUI elements
        self.camera_gui = arcade.Camera2D()

        # The scene which helps draw multiple spritelists in order.
        self.scene = self.create_scene()

        # Set up the player, specifically placing it at these coordinates.
        self.player_sprite = arcade.Sprite(
            playerfile,
            scale=CHARACTER_SCALING,   
        )
        
        #moving platforms
        self.one_platforms = self.tile_map.sprite_lists["1 tile moving platforms"]
        self.one2_platforms = self.tile_map.sprite_lists["1_tile moving platforms"]
      
    
        #enemies
        self.enemy_list = arcade.SpriteList()
        enemy = os.path.join(gameprogramming, "assets", "kenney_pixel-platformer", "Tiles", "Characters", "tile_0015.png")
        self.enemy_sprite = arcade.Sprite(enemy, CHARACTER_SCALING)
        #starting position
        enemy_spawn_point = self.scene["enemypoint"][0]
        self.enemy_sprite.center_x = enemy_spawn_point.center_x
        #have enemy spawn above enemypoint layer
        self.enemy_sprite.center_y = enemy_spawn_point.center_y + 32  
        self.enemy_list.append(self.enemy_sprite)

        self.enemy_sprite.change_x = 2.0  #speed
    
        
        # physics engine
        platforms = self.tile_map.sprite_lists["still platforms"], self.tile_map.sprite_lists["ground"],
        ladder = self.tile_map.sprite_lists["ladder"]
        moving_coins = self.tile_map.sprite_lists["coins"],  self.tile_map.sprite_lists["1_coins"]
        still_coins = self.tile_map.sprite_lists["stillcoins"]
        moving_platforms = [self.tile_map.sprite_lists["1 tile moving platforms"],  self.tile_map.sprite_lists["1_tile moving platforms"]]

        self.physics_engine = arcade.PhysicsEnginePlatformer(
            self.player_sprite, gravity_constant=GRAVITY, walls= platforms, ladders=ladder, platforms= moving_platforms)
        
        #moving the platforms
        self.start_position = []
        self.distances = [100, 200, 300]
        self.speed = [2.5, 1.0, -3.0]
        self.moving_platforms = moving_platforms
        #get start position of each moving platform and start moving platform
        for i in range(len(self.start_position)): 
            spritelist = self.moving_platforms[i]
            self.start_position.append(spritelist[0].position)

            for t in spritelist:
                    t.change_x = self.speed[i]


        # Keep track of the score
        self.score = 0

        # What key is pressed down?
        self.left_key_down = False
        self.right_key_down = False

        # Text object to display the score
        self.score_display = arcade.Text(
            "Points: 0",
            x=30,
            y=8,
            color=arcade.csscolor.WHITE,
            font_size=18,
        )
       
        self.reset()
        self.game_won = False

    def create_scene(self) -> arcade.Scene:
        """Load the tilemap and create the scene object."""
        # Our TileMap Object
        # Layer specific options are defined based on Layer names in a dictionary
        # Doing this will make the SpriteList for the platforms layer
        # use spatial hashing for collision detection.
         
        self.tile_map = arcade.TileMap(
            finalproject,
            layer_options= {
                "1 tile moving platforms" : {"use_spatial_hash": False},
                "1_tile moving platforms" : {"use_spatial_hash": False}
                    # "3 tile moving platorms" : {"use_spatial_hash": False}
               # "4 tile moving platorms" : {"use_spatial_hash": False}
            },

             scaling=TILE_SCALING,
            
        )
        # Set the window background 
        #https://api.arcade.academy/en/2.6.16/api_docs/api/window.html
        background = os.path.join(gameprogramming, "gpfinal", "menubackground", "cloudsbackground.jpeg")
        self.background = arcade.load_texture(background)

        
        #arcade.set_background_color(arcade.color.COLUMBIA_BLUE)
        # Use the tilemap's size to correctly set the camera's bounds.
        # Because how how shallow the map is we don't offset the bounds height
        self.camera_bounds = arcade.LRBT(
            self.window.width/2.0,
            self.tile_map.width * GRID_PIXEL_SIZE - self.window.width/2.0,
            self.window.height/2.0,
            self.tile_map.height * GRID_PIXEL_SIZE
        )

        print("Tilemap layers:", self.tile_map.sprite_lists.keys())

        # Our Scene Object
        # Initialize Scene with our TileMap, this will automatically add all layers
        # from the map as SpriteLists in the scene in the proper order.
        return arcade.Scene.from_tilemap(self.tile_map)

    def reset(self):
        """Reset the game to the initial state."""
        self.score = 0
        # Load a fresh scene to get the coins back
        self.scene = self.create_scene()

        # Move the player to start position
        self.player_sprite.position = self.tile_map.sprite_lists["flag spawnpoint"][0].position
        # Add the player to the scene
        self.scene.add_sprite("Player", self.player_sprite)
        self.game_won = False
    
        #move clouds
        self.cloud_list = self.scene["1 tile moving platforms"]
        for cloud in self.cloud_list:
            cloud.change_x = 0.5

        self.cloud2_list = self.scene["1_tile moving platforms"]
        for cloud in self.cloud2_list:  
            cloud.change_x = -0.5

        # Move coins 
        self.coin_list1 = self.scene["coins"]
        for coin in self.coin_list1:
            coin.change_x = -0.5
            coin.change_angle = 5

        self.coin_list2 = self.scene["1_coins"]
        for coin in self.coin_list2:
            coin.change_x = 0.5
            coin.change_angle = 10

        #enemies
        self.enemy_sprite.change_x = 2.0  
        self.scene.add_sprite("Enemy", self.enemy_sprite)

        self.physics_engine = arcade.PhysicsEnginePlatformer(
            self.player_sprite,
            gravity_constant=GRAVITY,
            walls=(self.scene["ground"], self.scene["still platforms"]),
            ladders=self.scene["ladder"],
            platforms=[self.scene["1 tile moving platforms"], self.scene["1_tile moving platforms"]]
        )
        self.enemy_physics = arcade.PhysicsEnginePlatformer(
            self.enemy_sprite,
            walls=self.scene["ground"],
            gravity_constant=GRAVITY,
        )   

    def on_draw(self):
        """Render the screen."""

        # Clear the screen to the background color
        self.clear()
        #self.enemy_list.draw()

        arcade.draw_texture_rect(self.background, 
                                 arcade.LBWH(0, 0, self.width, self.height) 
                                )
        # Draw the map with the sprite camera
        with self.camera_sprites.activate():
            # Draw our Scene
            # Note, if you a want pixelated look, add pixelated=True to the parameters
            self.scene.draw()

        # Draw the score with the gui camera
        with self.camera_gui.activate():
            # Draw our score on the screen. The camera keeps it in place.
            self.score_display.text = f"Score: {self.score}"
            self.score_display.draw()

        if self.game_won:
            arcade.draw_text("Game Over!", self.window.width / 2, self.window.height / 2,
                     arcade.color.BLUE_SAPPHIRE, font_name= "Courier New", font_size=50, anchor_x="center", anchor_y="center")

    def update_player_speed(self):
        # Calculate speed based on the keys pressed
        self.player_sprite.change_x = 0

        if self.left_key_down and not self.right_key_down:
            self.player_sprite.change_x = -PLAYER_MOVEMENT_SPEED
        elif self.right_key_down and not self.left_key_down:
            self.player_sprite.change_x = PLAYER_MOVEMENT_SPEED

    def on_key_press(self, key, modifiers):
        """Called whenever a key is pressed."""

        # Jump
        if key == arcade.key.UP or key == arcade.key.W:
            if self.physics_engine.can_jump():
                self.player_sprite.change_y = PLAYER_JUMP_SPEED

        # Left
        elif key == arcade.key.LEFT or key == arcade.key.A:
            self.left_key_down = True
            self.update_player_speed()

        # Right
        elif key == arcade.key.RIGHT or key == arcade.key.D:
            self.right_key_down = True
            self.update_player_speed()

        #climbing up ladder
        elif key == arcade.key.UP or key == arcade.key.W:
            if self.physics_engine.is_on_ladders():
                self.player_sprite.change_y = PLAYER_MOVEMENT_SPEED  
        elif self.physics_engine.can_jump():
                self.player_sprite.change_y = PLAYER_JUMP_SPEED       

        #climbing down ladder
        elif key == arcade.key.DOWN or key == arcade.key.S:
            if self.physics_engine.is_on_ladders():
                self.player_sprite.change_y = -PLAYER_MOVEMENT_SPEED  # Climb down


    def on_key_release(self, key, modifiers):
        """Called when the user releases a key."""
        if key == arcade.key.LEFT or key == arcade.key.A:
            self.left_key_down = False
            self.update_player_speed()
        elif key == arcade.key.RIGHT or key == arcade.key.D:
            self.right_key_down = False
            self.update_player_speed()

    def center_camera_to_player(self):
        # Move the camera to center on the player
        self.camera_sprites.position = arcade.math.smerp_2d(
            self.camera_sprites.position,
            self.player_sprite.position,
            self.window.delta_time,
            FOLLOW_DECAY_CONST,
        )

        # Constrain the camera's position to the camera bounds.
        self.camera_sprites.view_data.position = arcade.camera.grips.constrain_xy(
            self.camera_sprites.view_data, self.camera_bounds
        )

    def on_update(self, delta_time: float):
        """Movement and game logic"""

        # check for coins
        coin_hit_list= arcade.check_for_collision_with_lists(
            self.player_sprite, [self.scene["coins"],  self.scene["1_coins"],  self.scene["stillcoins"]]
        )

        # Loop through each coin we hit 
        for coin in coin_hit_list:
            # Remove the coin
            coin.remove_from_sprite_lists()
            # Add one to the score
            self.score += 1

        # Reverse direction at water and end flag 
        x_pos = self.scene["enemypoint"][0].center_x
        width = self.scene["enemypoint"][0].width
        if self.enemy_sprite.center_x < x_pos-width*3 or self.enemy_sprite.center_x > x_pos+width*7:
            self.enemy_sprite.change_x *= -1
        

        #collision check with enemies
        if arcade.check_for_collision(self.player_sprite, self.enemy_sprite):
            self.player_sprite.position = self.tile_map.sprite_lists["flag spawnpoint"][0].position

        #jump pad
        jump_booster_list = arcade.check_for_collision_with_list(
            self.player_sprite, self.scene["booster"]
        )

        #get player to jump in air after colliding with jump pad
        if jump_booster_list:
            self.player_sprite.change_y = PLAYER_JUMP_SPEED * 1.5

        # Move clouds
                # Get the x-positions of the flag spawnpoint and end flag
        spawn_x = self.tile_map.sprite_lists["flag spawnpoint"][0].center_x
        end_flag_x = self.tile_map.sprite_lists["end flag"][0].center_x

        # Reverse direction if clouds pass either flag
        def update_clouds(cloud_list):
            for cloud in cloud_list:
                #cloud.center_x += cloud.change_x

                # Reverse direction if past spawn or end flag
                if cloud.center_x > max(spawn_x, end_flag_x) or cloud.center_x < min(spawn_x, end_flag_x):
                    cloud.change_x *= -1

        update_clouds([*self.cloud_list, *self.coin_list1,  *self.coin_list2, *self.cloud2_list]) #unpacking all elements and putting into list 
        
            
        #check if player collides with water
        water_hit_list = arcade.check_for_collision_with_list (
            self.player_sprite, self.scene["water"]
        )

        #reset player if they collide with water 
        for water in water_hit_list:
            #reset player to flag spawnpoint
            self.reset()
        
        # Position the camera
        self.center_camera_to_player()

        if not self.game_won:
            flag_hit_list = arcade.check_for_collision_with_list(
                self.player_sprite, self.scene["end flag"]
            )
        if flag_hit_list:
            self.game_won = True
        
        self.physics_engine.update()
        # Move the player with the physics engine
        self.enemy_physics.update()

        self.coin_list1.update()
        self.coin_list2.update()
            
    def on_resize(self, width: int, height: int):
        """ Resize window """
        super().on_resize(width, height)
        # Update the cameras to match the new window size
        self.camera_sprites.match_window()
        # The position argument keeps `0, 0` in the bottom left corner.
        self.camera_gui.match_window(position=True)

def main():
    """Main function"""
    window = arcade.Window(WINDOW_WIDTH, WINDOW_HEIGHT, WINDOW_TITLE)
    
    menu = MenuView()
    window.show_view(menu)
    arcade.run()


if __name__ == "__main__":
    main()
