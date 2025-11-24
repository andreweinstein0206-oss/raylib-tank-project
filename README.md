float)GetRandomValue(60, screenHeight-60)};
    }

    answeringQuestion = false;
    currentPowerUpIndex = -1;
    answeringPlayer = 0;
    answerInput[0] = '\0';
    answerLength = 0;
    answerTimer = 0.0f;
}

static void spawnBullet(Vector2 pos, float rotDeg, int owner) {
    for (int i = 0; i < MAX_BULLETS; i++) {
        if (!bullets[i].active) {
            bullets[i].active = true;
            bullets[i].pos = pos;
            float rad = rotDeg * DEG2RAD;
            float speed = 700.0f;
            bullets[i].vel.x = cosf(rad) * speed;
            bullets[i].vel.y = sinf(rad) * speed;
            bullets[i].owner = owner;
            return;
        }
    }
}

void drawTankTexture(Tank *t, Texture2D texture) {
    float scale = 0.4f;
    float w = texture.width * scale;
    float h = texture.height * scale;
    Rectangle src = {0,0,texture.width,texture.height};
    Rectangle dest = {t->pos.x, t->pos.y, w, h};
    Vector2 origin = {w/2, h/2};
    DrawTexturePro(texture, src, dest, origin, t->rotation, WHITE);

    float barrelLength = 30.0f;
    float rad = t->rotation * DEG2RAD;
    Vector2 end = { t->pos.x + cosf(rad)*barrelLength, t->pos.y + sinf(rad)*barrelLength };
    DrawLineEx(t->pos, end, 1, BLACK);
}

// -------------------- MAP --------------------
void InitMap() {
    wallCount = 0;
    // Borders
    AddWall(0,0,1750,10,WHITE); AddWall(0,790,1750,10,WHITE);
    AddWall(0,0,10,800,WHITE); AddWall(1740,0,10,800,WHITE);

    // Sample obstacles
    AddWall(805,400,150,10,WHITE);
    AddWall(450,100,850,10,WHITE);
    AddWall(450,700,850,10,WHITE);
    AddWall(300,0,10,500,WHITE);
    AddWall(530,300,10,290,WHITE);
    AddWall(1200,230,10,290,WHITE);
    AddWall(1450,300,10,500,WHITE);
    AddWall(955,320,10,290,WHITE);
    AddWall(800,200,10,290,WHITE);
    AddWall(300,700,10,100,WHITE);
    AddWall(1450,0,10,100,WHITE);
}

// -------------------- MAIN --------------------
int main(void) {
    InitWindow(screenWidth, screenHeight, "Tank Game with Powerups");
    SetTargetFPS(60);
    InitAudioDevice();

    Texture2D tankBlue = LoadTexture("Graphics/BLUE.png");
    Texture2D tankRed  = LoadTexture("Graphics/RED.png");
    Texture2D bg       = LoadTexture("Graphics/background.png");
    Sound fxButton     = LoadSound("Graphics/buttonfx.wav");
    Sound bgMusic      = LoadSound("Graphics/park.wav");
    Sound shoot        = LoadSound("Graphics/shoot.wav");
    Sound power        = LoadSound("Graphics/power.wav");
    Sound fail         = LoadSound("Graphics/fail.wav");
    Sound down         = LoadSound("Graphics/down.wav");
    PlaySound(bgMusic);

    Rectangle startButton = { screenWidth/2-150, 300, 300, 80 };
    Rectangle exitButton  = { screenWidth/2-150, 450, 300, 80 };
    Color startColor = DARKGREEN, exitColor = MAROON;
    bool inMenu = true, running = true;

    float fireCooldown1 = 0.0f, fireCooldown2 = 0.0f;
    const float FIRE_COOLDOWN_BASE = 0.30f;
    float fireCooldownMax1 = FIRE_COOLDOWN_BASE;
    float fireCooldownMax2 = FIRE_COOLDOWN_BASE;
    const float TANK_SPEED_BASE = 200.0f;
    const float TANK_ROT_SPEED = 160.0f;

    resetGame();
    InitMap();

    while (running && !WindowShouldClose()) {
        Vector2 mousePoint = GetMousePosition();
        startColor = CheckCollisionPointRec(mousePoint, startButton) ? GREEN : DARKGREEN;
        exitColor  = CheckCollisionPointRec(mousePoint, exitButton) ? RED : MAROON;

        if (inMenu) {
            if (CheckCollisionPointRec(mousePoint, startButton) && IsMouseButtonPressed(MOUSE_LEFT_BUTTON)) {
                inMenu = false;
                resetGame();
                InitMap();
                PlaySound(fxButton);
                StopSound(bgMusic);
            }
            if (CheckCollisionPointRec(mousePoint, exitButton) && IsMouseButtonPressed(MOUSE_LEFT_BUTTON)) {
                running = false;
                break;
            }
            BeginDrawing();
            ClearBackground(RAYWHITE);
            if (bg.id!=0) DrawTexture(bg,0,0,WHITE);
            DrawText("ANDREW AND FRIENDS", screenWidth/2 - MeasureText("ANDREW AND FRIENDS",80)/2,200,80,BLACK);
            DrawRectangleRec(startButton,startColor);
            DrawRectangleRec(exitButton,exitColor);
            DrawText("START", startButton.x+90,startButton.y+25,40,WHITE);
            DrawText("EXIT", exitButton.x+115, exitButton.y+25,40,WHITE);
            EndDrawing();
            continue;
        }

        // -------------------- GAME UPDATE --------------------
        float dt = GetFrameTime();

        // Update timers


        if (rapidTimer1>0) { rapidTimer1-=dt; if(rapidTimer1<=0){rapidFireModifier1=1.0f; bulletBounceActive1=false;} }
        if (rapidTimer2>0) { rapidTimer2-=dt; if(rapidTimer2<=0){rapidFireModifier2=1.0f; bulletBounceActive2=false;} }



        // If question is active, handle its timing + input and skip movement/firing
        if (answeringQuestion) {
            answerTimer -= dt;
            if (answerTimer <= 0.0f) {
                // timeout => negative effect: -1 HP to answering player
                if (answeringPlayer == 1) tank1.hp = (tank1.hp>0) ? tank1.hp - 1 : 0;
                else if (answeringPlayer == 2) tank2.hp = (tank2.hp>0) ? tank2.hp - 1 : 0;

                // deactivate powerup and close question
                if (currentPowerUpIndex >= 0 && currentPowerUpIndex < MAX_POWERUPS)
                    powerups[currentPowerUpIndex].active = false;

                answeringQuestion = false;
                currentPowerUpIndex = -1;
                answeringPlayer = 0;
                answerInput[0] = '\0';
                answerLength = 0;
            }

            // capture typed characters (digits)
            int c = GetCharPressed(); // returns unicode char
            while (c > 0) {
                if ((c >= '0' && c <= '9') && answerLength < (int)(sizeof(answerInput)-1)) {
                    answerInput[answerLength++] = (char)c;
                    answerInput[answerLength] = '\0';
                }
                c = GetCharPressed();
            }

            if (IsKeyPressed(KEY_BACKSPACE) && answerLength > 0) {
                answerLength--;
                answerInput[answerLength] = '\0';
            }

            if (IsKeyPressed(KEY_ENTER) || IsKeyPressed(KEY_KP_ENTER)) {
                int typed = atoi(answerInput);
                bool correct = (typed == currentAnswer);

                if (correct) {
                    int type = powerups[currentPowerUpIndex].effectType;
                    if (answeringPlayer == 1) {
                        if (type == 1) { tank1.hp = (tank1.hp < 10) ? tank1.hp + 1 : tank1.hp; } // heal +1 (cap arbitrary)

                        if (type == 3) { rapidFireModifier1 = 0.5f; rapidTimer1 = 5.0f; } // faster fire for 5s
                        PlaySound(power);
                    } else {
                        if (type == 1) { tank2.hp = (tank2.hp < 10) ? tank2.hp + 1 : tank2.hp; }

                        if (type == 3) { rapidFireModifier2 = 0.5f; rapidTimer2 = 5.0f; }
                        PlaySound(power);
                    }
                } else {
                    // wrong answer -> -1 HP
                    if (answeringPlayer == 1) tank1.hp = (tank1.hp>0) ? tank1.hp - 1 : 0;
                    else if (answeringPlayer == 2) tank2.hp = (tank2.hp>0) ? tank2.hp - 1 : 0;
                    PlaySound(fail);
                }

                if (currentPowerUpIndex >= 0 && currentPowerUpIndex < MAX_POWERUPS)
                    powerups[currentPowerUpIndex].active = false;

                answeringQuestion = false;
                currentPowerUpIndex = -1;
                answeringPlayer = 0;
                answerInput[0] = '\0';
                answerLength = 0;
            }

            // While answering, skip the rest of the game update for movement/firing
        } else {
            // Player 1 movement
            float tankSpeed1 = TANK_SPEED_BASE;
            if (IsKeyDown(KEY_A)) tank1.rotation -= TANK_ROT_SPEED*dt;
            if (IsKeyDown(KEY_D)) tank1.rotation += TANK_ROT_SPEED*dt;
            if (IsKeyDown(KEY_W)) {
                tank1.pos.x += cosf(tank1.rotation*DEG2RAD)*tankSpeed1*dt;
                tank1.pos.y += sinf(tank1.rotation*DEG2RAD)*tankSpeed1*dt;
            }
            if (IsKeyDown(KEY_S)) {
                tank1.pos.x -= cosf(tank1.rotation*DEG2RAD)*tankSpeed1*dt*0.6f;
                tank1.pos.y -= sinf(tank1.rotation*DEG2RAD)*tankSpeed1*dt*0.6f;
            }

            // Player 2 movement
            float tankSpeed2 = TANK_SPEED_BASE;
            if (IsKeyDown(KEY_LEFT))  tank2.rotation -= TANK_ROT_SPEED*dt;
            if (IsKeyDown(KEY_RIGHT)) tank2.rotation += TANK_ROT_SPEED*dt;
            if (IsKeyDown(KEY_UP)) {
                tank2.pos.x += cosf(tank2.rotation*DEG2RAD)*tankSpeed2*dt;
                tank2.pos.y += sinf(tank2.rotation*DEG2RAD)*tankSpeed2*dt;
            }
            if (IsKeyDown(KEY_DOWN)) {
                tank2.pos.x -= cosf(tank2.rotation*DEG2RAD)*tankSpeed2*dt*0.6f;
                tank2.pos.y -= sinf(tank2.rotation*DEG2RAD)*tankSpeed2*dt*0.6f;
            }

            // Player picks up a power up (only when not currently answering)
            for (int i = 0; i < MAX_POWERUPS; i++) {
                    PlaySound(down);
                if (!powerups[i].active) continue;
                float dx1 = tank1.pos.x - powerups[i].pos.x;
                float dy1 = tank1.pos.y - powerups[i].pos.y;
                float dx2 = tank2.pos.x - powerups[i].pos.x;
                float dy2 = tank2.pos.y - powerups[i].pos.y;

                if (dx1*dx1 + dy1*dy1 < 40*40) {
                    answeringQuestion = true;
                    currentPowerUpIndex = i;
                    answeringPlayer = 1;
                    answerInput[0] = '\0';
                    answerLength = 0;

                    operandA = GetRandomValue(1, 10);
                    operandB = GetRandomValue(1, 10);
                    currentAnswer = operandA + operandB;
                    answerTimer = ANSWER_TIME;
                    break;


                }
                if (dx2*dx2 + dy2*dy2 < 40*40) {
                    answeringQuestion = true;
                    currentPowerUpIndex = i;
                    answeringPlayer = 2;
                    answerInput[0] = '\0';
                    answerLength = 0;

                    operandA = GetRandomValue(1, 10);
                    operandB = GetRandomValue(1, 10);
                    currentAnswer = operandA + operandB;
                    answerTimer = ANSWER_TIME;
                    break;
                }
            }

            // Tank vs walls collision (keep tanks out)
            TankCollideWalls(&tank1);
            TankCollideWalls(&tank2);

            // Keep tanks inside screen bounds
            ClampVec2(&tank1.pos, 50, 50, screenWidth-50, screenHeight-50);
            ClampVec2(&tank2.pos, 50, 50, screenWidth-50, screenHeight-50);

            // firing cooldowns: reduce timers
            if (fireCooldown1 > 0.0f) fireCooldown1 -= dt;
            if (fireCooldown2 > 0.0f) fireCooldown2 -= dt;

            // Update max cooldowns based on rapid fire modifiers
            fireCooldownMax1 = FIRE_COOLDOWN_BASE * rapidFireModifier1;
            fireCooldownMax2 = FIRE_COOLDOWN_BASE * rapidFireModifier2;

            if (IsKeyPressed(KEY_C) && fireCooldown1 <= 0.0f) {
                Vector2 tip = {tank1.pos.x + cosf(tank1.rotation*DEG2RAD)*40,
                               tank1.pos.y + sinf(tank1.rotation*DEG2RAD)*40};
                spawnBullet(tip, tank1.rotation, 1);
                fireCooldown1 = fireCooldownMax1;
                PlaySound(shoot);
            }
            if (IsKeyPressed(KEY_RIGHT_CONTROL) && fireCooldown2 <= 0.0f) {
                Vector2 tip = {tank2.pos.x + cosf(tank2.rotation*DEG2RAD)*40,
                               tank2.pos.y + sinf(tank2.rotation*DEG2RAD)*40};
                spawnBullet(tip, tank2.rotation, 2);
                fireCooldown2 = fireCooldownMax2;
                PlaySound(shoot);
            }
        } // end else (not answering)

        // Update bullets
        for (int i = 0; i < MAX_BULLETS; i++) {
            if (!bullets[i].active) continue;
            bullets[i].pos.x += bullets[i].vel.x*dt;
            bullets[i].pos.y += bullets[i].vel.y*dt;

            bool canBounce = (bullets[i].owner == 1) ? bulletBounceActive1 : bulletBounceActive2;

for (int j = 0; j < wallCount; j++) {
    Rectangle w = walls[j].rect;
    if (CheckCollisionCircleRec(bullets[i].pos, bulletRadius, w)) {
        if (canBounce) {
            // bounce logic as before
            float closestX = fmaxf(w.x, fminf(bullets[i].pos.x, w.x + w.width));
            float closestY = fmaxf(w.y, fminf(bullets[i].pos.y, w.y + w.height));
            float dx = bullets[i].pos.x - closestX;
            float dy = bullets[i].pos.y - closestY;

            if (fabsf(dx) > fabsf(dy)) {
                bullets[i].vel.x *= -1;
                bullets[i].pos.x += bullets[i].vel.x*dt;
            } else {
                bullets[i].vel.y *= -1;
                bullets[i].pos.y += bullets[i].vel.y*dt;
            }
        } else {
            // bullet disappears normally
            bullets[i].active = false;

        }
    }
}


            // Bullet-tank collision
            Tank *target = (bullets[i].owner==1) ? &tank2 : &tank1;
            float dx = bullets[i].pos.x - target->pos.x;
            float dy = bullets[i].pos.y - target->pos.y;
            if (dx*dx + dy*dy <= (tankRadius + bulletRadius)*(tankRadius + bulletRadius)){
                target->hp--;
                bullets[i].active=false;
                if (bullets[i].owner==1) tank1.score++; else tank2.score++;
                if (target->hp<=0) { StopSound(down); inMenu=true; resetGame(); break;}
            }
        }

        if (IsKeyPressed(KEY_ESCAPE)) { inMenu=true; resetGame(); }

        // Draw
        BeginDrawing();
        ClearBackground(RAYWHITE);
        if (bg.id!=0) DrawTexture(bg,0,0,WHITE);

        // Draw walls
        DrawWalls();

        // Draw bullets
        for (int i = 0; i < MAX_BULLETS; i++) if (bullets[i].active) DrawCircleV(bullets[i].pos, bulletRadius, BLACK);

        // Draw powerups (under tanks)
        DrawPowerUps();

        // Draw tanks
        drawTankTexture(&tank1, tankBlue);
        drawTankTexture(&tank2, tankRed);

        // Draw healthbars
        DrawHealthBar(tank1.pos, tank1.hp, BLUE);
        DrawHealthBar(tank2.pos, tank2.hp, RED);

        // HUD
        DrawText(TextFormat("P1 HP: %d  Score: %d", tank1.hp, tank1.score), 20, 20, 20, WHITE);
        DrawText(TextFormat("P2 HP: %d  Score: %d", tank2.hp, tank2.score), screenWidth-260, 20, 20, WHITE);
        DrawText("Press ESC to return to menu", screenWidth/2-150, screenHeight-40, 20, WHITE);

        // If question overlay is active, draw it on top
        if (answeringQuestion) {
            DrawRectangle(0, 0, screenWidth, screenHeight, Fade(BLACK, 0.6f));
            DrawText(TextFormat("Player %d - Answer the question!", answeringPlayer), screenWidth/2 - 220, 160, 30, WHITE);
            DrawText(TextFormat("What is %d + %d ?", operandA, operandB), screenWidth/2 - 90, 220, 28, WHITE);
            DrawText(TextFormat("Your Answer: %s", answerInput), screenWidth/2 - 100, 280, 28, GREEN);
            DrawText(TextFormat("Time Left: %.1f", answerTimer), screenWidth/2 - 60, 340, 26, ORANGE);
            DrawText("Type digits, press ENTER to submit, BACKSPACE to erase", screenWidth/2 - 320, 400, 18, LIGHTGRAY);
        }

        EndDrawing();
    }

    // Cleanup
    if (bg.id!=0) UnloadTexture(bg);
    UnloadTexture(tankBlue);
    UnloadTexture(tankRed);
    UnloadSound(fxButton);
    UnloadSound(bgMusic);
    UnloadSound(shoot);
    UnloadSound(power);
    UnloadSound(fail);
    UnloadSound(down);
    CloseAudioDevice();
    CloseWindow();
    return 0;
}
