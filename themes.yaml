import javax.swing.*;
import javax.imageio.ImageIO;
import java.awt.*;
import java.awt.event.*;
import java.awt.image.BufferedImage;
import java.io.*;
import java.nio.file.Files;
import java.nio.file.Paths;
import java.util.ArrayList;
import java.util.Arrays;
import java.util.LinkedHashMap;
import java.util.List;
import java.util.Map;

public class tetris extends JFrame {
    private TetrisPanel gamePanel;

    public tetris() {
        setTitle("Tetris - Advanced");
        setDefaultCloseOperation(JFrame.EXIT_ON_CLOSE);
        setResizable(false);
        gamePanel = new TetrisPanel();
        add(gamePanel);
        pack();
        setLocationRelativeTo(null);
        setVisible(true);
    }

    public static void main(String[] args) {
        SwingUtilities.invokeLater(() -> new tetris());
    }
}

class TetrisPanel extends JPanel {
    private static final int WIDTH = 700;
    private static final int HEIGHT = 950;
    private static final int BLOCK_SIZE = 30;
    private static final int GRID_WIDTH = 10;
    private static final int GRID_HEIGHT = 30;
    private static final int PADDING = 40;
    private static final int GRID_OFFSET_X = 175;
    private static final int GRID_OFFSET_Y = 10;
    private static final int PREFERRED_WIDTH = GRID_OFFSET_X + (GRID_WIDTH * BLOCK_SIZE) + PADDING + 10;
    private static final String THEMES_FILE = "themes.yaml";

    private int[][] grid;
    private Tetromino currentTetromino;
    private Tetromino nextTetromino;
    private int score;
    private int lines;
    private int level;
    private boolean gameOver;
    private boolean gamePaused;
    private Timer gameTimer;
    private float renderPieceY;
    private float targetPieceY;
    private float renderPieceX;
    private float targetPieceX;
    private Timer renderTimer;

    private List<Integer> highScores;
    private static final String HIGH_SCORES_FILE = "highscores.txt";
    private List<Tetromino> holdQueue; 
    private Tetromino heldTetromino;
    private boolean canSwap;
    private int comboCount;
    private long gameOverImageTime = 0;
    private static final long IMAGE_DISPLAY_TIME = 2000;
    private BufferedImage gameOverImage = null;
    private boolean imageLoadAttempted = false;
    private int guiDesign = 0; // index into designNames -- also the theme key used to look up colors
    private Rectangle[] designButtons = new Rectangle[4];
    private String[] designNames = {"Default", "Sunset", "Retro", "Win95"};
    private static final int BUTTON_WIDTH = 80;
    private static final int BUTTON_HEIGHT = 30;
    private static final int BUTTONS_START_Y = 720;
    private static final int BUTTON_SPACING = 10;

    // Themes loaded from themes.yaml, keyed by theme name (matches entries in designNames)
    private Map<String, Theme> themes;

    public TetrisPanel() {
        setPreferredSize(new Dimension(PREFERRED_WIDTH, HEIGHT));
        setBackground(Color.BLACK);
        setFocusable(true);
        requestFocus();

        loadThemes();

        grid = new int[GRID_HEIGHT][GRID_WIDTH];
        score = 0;
        lines = 0;
        level = 1;
        gameOver = false;
        gamePaused = false;
        highScores = new ArrayList<>();
        holdQueue = new ArrayList<>();
        heldTetromino = null;
        canSwap = true;
        comboCount = 0;
        loadHighScores();

        currentTetromino = new Tetromino();
        nextTetromino = new Tetromino();
        
        // Initialize design buttons - centered
        int totalButtonWidth = 4 * BUTTON_WIDTH + 3 * BUTTON_SPACING;
        int startX = (PREFERRED_WIDTH - totalButtonWidth) / 2;
        for (int i = 0; i < 4; i++) {
            designButtons[i] = new Rectangle(startX + i * (BUTTON_WIDTH + BUTTON_SPACING), BUTTONS_START_Y, BUTTON_WIDTH, BUTTON_HEIGHT);
        }
        
        addMouseListener(new MouseAdapter() {
            @Override
            public void mouseClicked(MouseEvent e) {
                for (int i = 0; i < designButtons.length; i++) {
                    if (designButtons[i].contains(e.getPoint())) {
                        guiDesign = i;
                        repaint();
                    }
                }
            }
        });

        addKeyListener(new KeyAdapter() {
            @Override
            public void keyPressed(KeyEvent e) {
                if (e.getKeyCode() == KeyEvent.VK_P) {
                    gamePaused = !gamePaused;
                    repaint();
                    return;
                }
                
                if (e.getKeyCode() == KeyEvent.VK_D) {
                    guiDesign = (guiDesign + 1) % 4;
                    repaint();
                    return;
                }

                if (gamePaused) return;

                if (!gameOver) {
                    switch (e.getKeyCode()) {
                        case KeyEvent.VK_LEFT:
                            moveLeft();
                            break;
                        case KeyEvent.VK_RIGHT:
                            moveRight();
                            break;
                        case KeyEvent.VK_DOWN:
                            hardDrop();
                            break;
                        case KeyEvent.VK_UP:
                            rotate();
                            break;
                        case KeyEvent.VK_C:
                            holdPiece();
                            break;
                        case KeyEvent.VK_SPACE:
                            softDrop();
                            break;
                    }
                    repaint();
                } else {
                    if (e.getKeyCode() == KeyEvent.VK_R) {
                        restartGame();
                    }
                }
            }
        });

        gameTimer = new Timer(Math.max(100, 300 - (level * 30)), e -> {
            if (!gameOver && !gamePaused) {
                moveDown(false);
            }
        });
        gameTimer.start();

        renderPieceY = currentTetromino.y * BLOCK_SIZE;
        targetPieceY = renderPieceY;
        renderPieceX = currentTetromino.x * BLOCK_SIZE;
        targetPieceX = renderPieceX;
        renderTimer = new Timer(16, e -> {
            if (!gameOver && !gamePaused) {
                float frameDelay = renderTimer.getDelay();
                float gameDelay = Math.max(1, gameTimer.getDelay());
                float step = BLOCK_SIZE * (frameDelay / gameDelay);

                if (Math.abs(renderPieceY - targetPieceY) > 0.5f) {
                    if (renderPieceY < targetPieceY) {
                        renderPieceY = Math.min(renderPieceY + step, targetPieceY);
                    } else {
                        renderPieceY = Math.max(renderPieceY - step, targetPieceY);
                    }
                }
                if (Math.abs(renderPieceX - targetPieceX) > 0.5f) {
                    if (renderPieceX < targetPieceX) {
                        renderPieceX = Math.min(renderPieceX + step, targetPieceX);
                    } else {
                        renderPieceX = Math.max(renderPieceX - step, targetPieceX);
                    }
                }
            }
            repaint();
        });
        renderTimer.start();
    }

    /**
     * Loads all themes from themes.yaml (searched in the working directory).
     * If the file is missing, unreadable, or a design is not defined inside it,
     * a safe fallback theme is generated so the game never crashes because of it.
     */
    private void loadThemes() {
        themes = ThemeLoader.load(resolveThemesPath());

        for (String name : designNames) {
            if (!themes.containsKey(name)) {
                System.err.println("Warnung: Theme '" + name + "' wurde nicht in " + THEMES_FILE + " gefunden, verwende Standardwerte.");
                Theme fallback = new Theme();
                fallback.name = name;
                fallback.buttonColors = Arrays.asList(Color.GRAY, Color.GRAY, Color.GRAY, Color.GRAY);
                fallback.paletteColors = Arrays.asList(Color.RED, Color.GREEN, Color.BLUE, Color.YELLOW, Color.CYAN, Color.MAGENTA, Color.ORANGE);
                fallback.applyFallbacks();
                themes.put(name, fallback);
            }
        }
    }

    private String resolveThemesPath() {
        String[] candidates = {
            THEMES_FILE,
            System.getProperty("user.dir") + File.separator + THEMES_FILE
        };
        for (String c : candidates) {
            if (new File(c).exists()) {
                return c;
            }
        }
        return THEMES_FILE;
    }

    /** Returns the Theme object for the currently selected design. Never returns null. */
    private Theme currentTheme() {
        Theme t = themes.get(designNames[guiDesign]);
        if (t != null) return t;
        if (!themes.isEmpty()) return themes.values().iterator().next();
        Theme empty = new Theme();
        empty.applyFallbacks();
        return empty;
    }

    private Color withAlpha(Color c, int alpha) {
        return new Color(c.getRed(), c.getGreen(), c.getBlue(), alpha);
    }

    private void bild() {
        if(gameOver) {
            if (!imageLoadAttempted) {
                imageLoadAttempted = true;
                try {
                    File file = new File("0H:\\Desktop\\kimera-evo38.jpg");
                    gameOverImage = ImageIO.read(file);
                    if (gameOverImage != null) {
                        System.out.println("Bild erfolgreich geladen: " + gameOverImage.getWidth() + "x" + gameOverImage.getHeight());
                    } else {
                        System.err.println("Bild ist null nach ImageIO.read()");
                    }
                } catch (IOException e) {
                    System.err.println("Fehler beim Laden des Bildes: " + e.getMessage());
                    e.printStackTrace();
                }
            }
            gameOverImageTime = System.currentTimeMillis();
        }
    }

    private void holdPiece() {
        if (!canSwap) return;

        if (heldTetromino == null) {
            heldTetromino = currentTetromino;
            currentTetromino = nextTetromino;
            nextTetromino = new Tetromino();
        } else {
            Tetromino temp = currentTetromino;
            currentTetromino = heldTetromino;
            heldTetromino = temp;
        }
        currentTetromino.x = 3;
        currentTetromino.y = 0;
        canSwap = false;
        renderPieceY = currentTetromino.y * BLOCK_SIZE;
        targetPieceY = renderPieceY;
        renderPieceX = currentTetromino.x * BLOCK_SIZE;
        targetPieceX = renderPieceX;
    }

    private void softDrop() {
        if (canMove(currentTetromino.x, currentTetromino.y + 1, currentTetromino.shape)) {
            currentTetromino.y++;
            targetPieceY = currentTetromino.y * BLOCK_SIZE;
            score += 1;
        }
    }

    private void loadHighScores() {
        try (BufferedReader reader = new BufferedReader(new FileReader(HIGH_SCORES_FILE))) {
            String line;
            while ((line = reader.readLine()) != null) {
                try {
                    highScores.add(Integer.parseInt(line));
                } catch (NumberFormatException e) {}
            }
        } catch (IOException e) {}
    }

    private void saveHighScore(int s) {
        highScores.add(s);
        highScores.sort((a, b) -> b.compareTo(a));
        if (highScores.size() > 10) {
            highScores.remove(highScores.size() - 1);
        }
        try (PrintWriter writer = new PrintWriter(new FileWriter(HIGH_SCORES_FILE))) {
            for (int score : highScores) {
                writer.println(score);
            }
        } catch (IOException e) {
            e.printStackTrace();
        }
    }

    private void moveLeft() {
        if (canMove(currentTetromino.x - 1, currentTetromino.y, currentTetromino.shape)) {
            currentTetromino.x--;
            targetPieceX = currentTetromino.x * BLOCK_SIZE;
            renderPieceX = targetPieceX;
        }
    }

    private void moveRight() {
        if (canMove(currentTetromino.x + 1, currentTetromino.y, currentTetromino.shape)) {
            currentTetromino.x++;
            targetPieceX = currentTetromino.x * BLOCK_SIZE;
            renderPieceX = targetPieceX;
        }
    }

    private void moveDown(boolean snap) {
        if (canMove(currentTetromino.x, currentTetromino.y + 1, currentTetromino.shape)) {
            currentTetromino.y++;
            targetPieceY = currentTetromino.y * BLOCK_SIZE;
            if (snap) {
                renderPieceY = targetPieceY;
            }
        } else {
            placeTetromino();
            clearLines();
            spawnNewTetromino();
        }
    }

    private void hardDrop() {
        int dropDistance = 0;
        while (canMove(currentTetromino.x, currentTetromino.y + 1, currentTetromino.shape)) {
            currentTetromino.y++;
            dropDistance++;
        }
        score += dropDistance * 2;
        targetPieceY = currentTetromino.y * BLOCK_SIZE;
        renderPieceY = targetPieceY;
        targetPieceX = currentTetromino.x * BLOCK_SIZE;
        renderPieceX = targetPieceX;
        placeTetromino();
        clearLines();
        spawnNewTetromino();
    }

    private void rotate() {
        int[][] rotated = rotateShape(copyShape(currentTetromino.shape));
        if (canMove(currentTetromino.x, currentTetromino.y, rotated)) {
            currentTetromino.shape = rotated;
            targetPieceX = currentTetromino.x * BLOCK_SIZE;
            targetPieceY = currentTetromino.y * BLOCK_SIZE;
        }
    }

    private int[][] copyShape(int[][] shape) {
        int[][] copy = new int[shape.length][shape[0].length];
        for (int i = 0; i < shape.length; i++) {
            System.arraycopy(shape[i], 0, copy[i], 0, shape[i].length);
        }
        return copy;
    }

    private int[][] rotateShape(int[][] shape) {
        int rows = shape.length;
        int cols = shape[0].length;
        int[][] rotated = new int[cols][rows];
        for (int i = 0; i < rows; i++) {
            for (int j = 0; j < cols; j++) {
                rotated[j][rows - 1 - i] = shape[i][j];
            }
        }
        return rotated;
    }

    private boolean canMove(int x, int y, int[][] shape) {
        for (int i = 0; i < shape.length; i++) {
            for (int j = 0; j < shape[0].length; j++) {
                if (shape[i][j] == 1) {
                    int newX = x + j;
                    int newY = y + i;
                    if (newX < 0 || newX >= GRID_WIDTH || newY >= GRID_HEIGHT) {
                        return false;
                    }
                    if (newY >= 0 && grid[newY][newX] != 0) {
                        return false;
                    }
                }
            }
        }
        return true;
    }

    private void placeTetromino() {
        for (int i = 0; i < currentTetromino.shape.length; i++) {
            for (int j = 0; j < currentTetromino.shape[0].length; j++) {
                if (currentTetromino.shape[i][j] == 1) {
                    int x = currentTetromino.x + j;
                    int y = currentTetromino.y + i;
                    if (y >= 0 && y < GRID_HEIGHT && x >= 0 && x < GRID_WIDTH) {
                        grid[y][x] = currentTetromino.color;
                    } else if (y < 0) {
                        gameOver = true;
                    }
                }
            }
        }
        if (gameOver) {
            saveHighScore(score);
            loadHighScores();
            bild();
        }
    }

    private void clearLines() {
        int linesCleared = 0;
        for (int i = GRID_HEIGHT - 1; i >= 0; i--) {
            boolean fullLine = true;
            for (int j = 0; j < GRID_WIDTH; j++) {
                if (grid[i][j] == 0) {
                    fullLine = false;
                    break;
                }
            }
            if (fullLine) {
                linesCleared++;
                for (int k = i; k > 0; k--) {
                    System.arraycopy(grid[k - 1], 0, grid[k], 0, GRID_WIDTH);
                }
                System.arraycopy(new int[GRID_WIDTH], 0, grid[0], 0, GRID_WIDTH);
                i++;
            }
        }

        if (linesCleared > 0) {
            lines += linesCleared;
            comboCount++;
            int baseScore = linesCleared == 4 ? 800 : linesCleared * 100;
            score += baseScore * level * comboCount;
            level = Math.min(20, 1 + (lines / 10));
            gameTimer.setDelay(Math.max(100, 300 - (level * 30)));
        } else {
            comboCount = 0;
        }
    }

    private void spawnNewTetromino() {
        currentTetromino = nextTetromino;
        nextTetromino = new Tetromino();
        canSwap = true;
        renderPieceY = currentTetromino.y * BLOCK_SIZE;
        targetPieceY = renderPieceY;
        renderPieceX = currentTetromino.x * BLOCK_SIZE;
        targetPieceX = renderPieceX;
        if (!canMove(currentTetromino.x, currentTetromino.y, currentTetromino.shape)) {
            gameOver = true;
        }
    }

    private void restartGame() {
        grid = new int[GRID_HEIGHT][GRID_WIDTH];
        score = 0;
        lines = 0;
        level = 1;
        gameOver = false;
        gamePaused = false;
        heldTetromino = null;
        canSwap = true;
        comboCount = 0;
        gameOverImageTime = 0;
        currentTetromino = new Tetromino();
        nextTetromino = new Tetromino();
        renderPieceY = currentTetromino.y * BLOCK_SIZE;
        targetPieceY = renderPieceY;
        renderPieceX = currentTetromino.x * BLOCK_SIZE;
        targetPieceX = renderPieceX;
        gameTimer.setDelay(300);
    }

    @Override
    protected void paintComponent(Graphics g) {
        super.paintComponent(g);
        Graphics2D g2d = (Graphics2D) g;
        g2d.setRenderingHint(RenderingHints.KEY_ANTIALIASING, RenderingHints.VALUE_ANTIALIAS_ON);

        Theme theme = currentTheme();
        g2d.setColor(theme.background);
        g2d.fillRect(0, 0, WIDTH, HEIGHT);
        
        drawGrid(g2d);
        drawGhostPiece(g2d);
        drawTetromino(g2d, currentTetromino);
        drawScore(g2d);
        drawNextPiece(g2d);
        drawHeldPiece(g2d);
        drawStats(g2d);

        if (gamePaused) {
            drawPaused(g2d);
        }
        if (gameOver) {
            drawGameOver(g2d);
        }
    }

    private void drawStats(Graphics2D g) {
        Theme theme = currentTheme();
        Font font;
        if (guiDesign == 2) {
            font = new Font("Courier New", Font.PLAIN, 14);
        } else if (guiDesign == 3) {
            font = new Font("MS Sans Serif", Font.PLAIN, 11);
        } else {
            font = new Font("Arial", Font.BOLD, 14);
        }

        g.setFont(font);
        g.setColor(theme.stat1Color);
        g.drawString("Lines: " + lines, 10, 250);
        g.setColor(theme.stat2Color);
        g.drawString("Level: " + level, 10, 280);
        g.setColor(theme.stat3Color);
        g.drawString("Combo: " + comboCount, 10, 310);
    }

    private void drawPaused(Graphics2D g) {
        Theme theme = currentTheme();

        if (guiDesign == 0) {
            g.setColor(withAlpha(Color.BLACK, 150));
            g.fillRect(0, 0, WIDTH, HEIGHT);
            g.setColor(theme.scoreColor);
            g.setFont(new Font("Arial", Font.BOLD, 48));
            g.drawString("PAUSED", 250, 300);
            g.setColor(theme.labelColor);
            g.setFont(new Font("Arial", Font.BOLD, 16));
            g.drawString("Press P to Resume | Press D to Change Design", 220, 350);
        } else if (guiDesign == 1) {
            g.setColor(withAlpha(theme.background, 180));
            g.fillRect(0, 0, WIDTH, HEIGHT);
            g.setColor(withAlpha(theme.border, 150));
            g.fillRect(200, 260, 300, 100);
            g.setColor(theme.scoreColor);
            g.setFont(new Font("Arial", Font.BOLD, 48));
            g.drawString("PAUSED", 250, 320);
            g.setColor(theme.labelColor);
            g.setFont(new Font("Arial", Font.BOLD, 14));
            g.drawString(">> Press P to Resume | Press D to Change Design <<", 180, 380);
        } else if (guiDesign == 2) {
            g.setColor(withAlpha(Color.DARK_GRAY, 180));
            g.fillRect(0, 0, WIDTH, HEIGHT);
            g.setColor(theme.border);
            g.setStroke(new BasicStroke(3));
            g.drawRect(150, 250, 400, 150);
            g.setFont(new Font("Courier New", Font.BOLD, 40));
            g.setColor(theme.scoreColor);
            g.drawString("PAUSED", 280, 320);
            g.setColor(theme.labelColor);
            g.setFont(new Font("Courier New", Font.PLAIN, 14));
            g.drawString("[P] RESUME  [D] DESIGN", 260, 380);
        } else {
            g.setColor(withAlpha(theme.background, 200));
            g.fillRect(0, 0, WIDTH, HEIGHT);

            g.setColor(theme.buttonUnselectedLight);
            g.setStroke(new BasicStroke(2));
            g.drawRect(150, 200, 400, 250);
            g.setColor(theme.buttonUnselectedDark);
            g.drawLine(151, 200, 151, 450);
            g.drawLine(150, 200, 550, 200);

            g.setColor(theme.border);
            g.fillRect(150, 200, 400, 25);
            g.setColor(theme.buttonTextColor);
            g.setFont(new Font("MS Sans Serif", Font.BOLD, 14));
            g.drawString("PAUSED", 300, 220);

            g.setColor(theme.labelColor);
            g.setFont(new Font("MS Sans Serif", Font.PLAIN, 12));
            g.drawString("P = Resume  |  D = Design", 220, 300);
        }
    }

    /** Looks up the color for a placed/falling piece from the current theme's palette. */
    private Color getDesignColor(int colorIndex) {
        return currentTheme().paletteColor(colorIndex);
    }

    private void drawGrid(Graphics2D g) {
        Theme theme = currentTheme();
        Color bgColor = theme.background;
        Color gridColor = theme.grid;
        Color borderColor = theme.border;

        g.setColor(bgColor);
        g.fillRect(GRID_OFFSET_X - 2, GRID_OFFSET_Y - 2, GRID_WIDTH * BLOCK_SIZE + 4, GRID_HEIGHT * BLOCK_SIZE + 4);

        g.setColor(gridColor);
        int dotSize = Math.max(2, BLOCK_SIZE / 8);
        for (int i = 0; i < GRID_HEIGHT; i++) {
            for (int j = 0; j < GRID_WIDTH; j++) {
                int cx = GRID_OFFSET_X + j * BLOCK_SIZE + BLOCK_SIZE / 2 - dotSize / 2;
                int cy = GRID_OFFSET_Y + i * BLOCK_SIZE + BLOCK_SIZE / 2 - dotSize / 2;
                g.fillRect(cx, cy, dotSize, dotSize);
            }
        }

        for (int i = 0; i < GRID_HEIGHT; i++) {
            for (int j = 0; j < GRID_WIDTH; j++) {
                if (grid[i][j] != 0) {
                    Color blockColor = getDesignColor(grid[i][j]);
                    drawBlockAtCell(g, j, i, blockColor);
                } else {
                    g.setColor(withAlpha(gridColor, 130));
                    g.drawRect(GRID_OFFSET_X + j * BLOCK_SIZE, GRID_OFFSET_Y + i * BLOCK_SIZE, BLOCK_SIZE, BLOCK_SIZE);
                }
            }
        }

        g.setFont(new Font(Font.MONOSPACED, Font.BOLD, Math.max(12, BLOCK_SIZE / 2)));
        g.setColor(borderColor);
        for (int i = 0; i < GRID_HEIGHT; i++) {
            int y = GRID_OFFSET_Y + i * BLOCK_SIZE + BLOCK_SIZE / 2 + 6;
            g.drawString("<", GRID_OFFSET_X - 28, y);
            g.drawString(">", GRID_OFFSET_X + GRID_WIDTH * BLOCK_SIZE + 16, y);
        }
        
        // Draw border with Windows 95 beveled effect for Win95 design
        if (guiDesign == 3) {
            g.setColor(theme.buttonUnselectedLight);
            g.setStroke(new BasicStroke(2));
            g.drawRect(GRID_OFFSET_X - 3, GRID_OFFSET_Y - 3, GRID_WIDTH * BLOCK_SIZE + 6, GRID_HEIGHT * BLOCK_SIZE + 6);
            g.setColor(theme.buttonUnselectedDark);
            g.drawRect(GRID_OFFSET_X - 2, GRID_OFFSET_Y - 2, GRID_WIDTH * BLOCK_SIZE + 4, GRID_HEIGHT * BLOCK_SIZE + 4);
        } else {
            g.setColor(borderColor);
            g.setStroke(new BasicStroke(2));
            g.drawRect(GRID_OFFSET_X - 2, GRID_OFFSET_Y - 2, GRID_WIDTH * BLOCK_SIZE + 4, GRID_HEIGHT * BLOCK_SIZE + 4);
        }
    }

    private void drawTetromino(Graphics2D g, Tetromino t) {
        for (int i = 0; i < t.shape.length; i++) {
            for (int j = 0; j < t.shape[0].length; j++) {
                if (t.shape[i][j] == 1) {
                    int x = GRID_OFFSET_X + (int) renderPieceX + j * BLOCK_SIZE;
                    int y = GRID_OFFSET_Y + (int) renderPieceY + i * BLOCK_SIZE;
                    Color blockColor = getDesignColor(t.color);
                    drawBlockAtPixel(g, x, y, blockColor);
                }
            }
        }
    }

    private void drawGhostPiece(Graphics2D g) {
        // Calculate where the piece will land
        Tetromino ghost = new Tetromino();
        ghost.shape = copyShape(currentTetromino.shape);
        ghost.x = currentTetromino.x;
        ghost.y = currentTetromino.y;

        // Drop the ghost piece to the bottom
        while (canMove(ghost.x, ghost.y + 1, ghost.shape)) {
            ghost.y++;
        }

        // Draw ghost piece with semi-transparency
        for (int i = 0; i < ghost.shape.length; i++) {
            for (int j = 0; j < ghost.shape[0].length; j++) {
                if (ghost.shape[i][j] == 1) {
                    int cellX = ghost.x + j;
                    int cellY = ghost.y + i;

                    if (cellX >= 0 && cellX < GRID_WIDTH && cellY >= 0 && cellY < GRID_HEIGHT) {
                        int x = GRID_OFFSET_X + cellX * BLOCK_SIZE;
                        int y = GRID_OFFSET_Y + cellY * BLOCK_SIZE;

                        // Draw semi-transparent ghost with design-aware color
                        Color baseColor = getDesignColor(currentTetromino.color);
                        Color ghostColor = new Color(baseColor.getRed(), baseColor.getGreen(), baseColor.getBlue(), 70);
                        g.setColor(ghostColor);
                        g.fillRect(x, y, BLOCK_SIZE, BLOCK_SIZE);

                        g.setColor(new Color(baseColor.getRed(), baseColor.getGreen(), baseColor.getBlue(), 100));
                        g.setStroke(new BasicStroke(2));
                        g.drawRect(x, y, BLOCK_SIZE, BLOCK_SIZE);
                    }
                }
            }
        }
    }

    private void drawBlockAtCell(Graphics2D g, int cellX, int cellY, Color base) {
        int x = GRID_OFFSET_X + cellX * BLOCK_SIZE;
        int y = GRID_OFFSET_Y + cellY * BLOCK_SIZE;
        drawBlockAtPixel(g, x, y, base);
    }

    private void drawBlockAtPixel(Graphics2D g, int x, int y, Color base) {
        Color darker = base.darker();
        Color brighter = base.brighter();

        g.setColor(darker);
        g.fillRect(x + 1, y + 1, BLOCK_SIZE - 2, BLOCK_SIZE - 2);

        int pad = Math.max(2, BLOCK_SIZE / 6);
        g.setColor(brighter);
        g.fillRect(x + pad, y + pad, BLOCK_SIZE - pad * 2, BLOCK_SIZE - pad * 2);

        g.setColor(new Color(0, 0, 0, 120));
        g.drawRect(x + 1, y + 1, BLOCK_SIZE - 2, BLOCK_SIZE - 2);

        int dot = Math.max(2, BLOCK_SIZE / 8);
        g.setColor(darker.darker());
        g.fillRect(x + BLOCK_SIZE / 2 - dot / 2, y + BLOCK_SIZE / 2 - dot / 2, dot, dot);
    }

    private void drawDesignButtons(Graphics2D g) {
        Theme theme = currentTheme();
        for (int i = 0; i < designButtons.length; i++) {
            Rectangle btn = designButtons[i];
            boolean isSelected = (i == guiDesign);
            Color btnColor = theme.buttonColor(i);

            if (guiDesign == 3) {
                // Windows 95 style buttons
                g.setColor(btnColor);
                g.fillRect(btn.x, btn.y, btn.width, btn.height);

                g.setColor(isSelected ? theme.buttonSelectedLight : theme.buttonUnselectedLight);
                g.drawLine(btn.x, btn.y, btn.x + btn.width, btn.y);
                g.drawLine(btn.x, btn.y, btn.x, btn.y + btn.height);

                g.setColor(isSelected ? theme.buttonSelectedDark : theme.buttonUnselectedDark);
                g.drawLine(btn.x + btn.width, btn.y, btn.x + btn.width, btn.y + btn.height);
                g.drawLine(btn.x, btn.y + btn.height, btn.x + btn.width, btn.y + btn.height);

                g.setColor(theme.buttonTextColor);
                g.setFont(new Font("MS Sans Serif", Font.PLAIN, 10));
                FontMetrics fm = g.getFontMetrics();
                int textX = btn.x + (btn.width - fm.stringWidth(designNames[i])) / 2;
                int textY = btn.y + ((btn.height - fm.getHeight()) / 2) + fm.getAscent();
                g.drawString(designNames[i], textX, textY);
            } else {
                // Colorful buttons for other designs
                g.setColor(btnColor);
                g.fillRect(btn.x, btn.y, btn.width, btn.height);

                if (isSelected) {
                    g.setStroke(new BasicStroke(3));
                    g.setColor(theme.buttonSelectedLight);
                } else {
                    g.setStroke(new BasicStroke(1));
                    g.setColor(theme.buttonUnselectedDark);
                }
                g.drawRect(btn.x, btn.y, btn.width, btn.height);

                g.setColor(theme.buttonTextColor);
                g.setFont(new Font("Arial", Font.BOLD, 9));
                FontMetrics fm = g.getFontMetrics();
                int textX = btn.x + (btn.width - fm.stringWidth(designNames[i])) / 2;
                int textY = btn.y + ((btn.height - fm.getHeight()) / 2) + fm.getAscent();
                g.drawString(designNames[i], textX, textY);
            }
        }
    }

    private void drawScore(Graphics2D g) {
        Theme theme = currentTheme();
        Font scoreFont;
        if (guiDesign == 2) {
            scoreFont = new Font("Courier New", Font.BOLD, 20);
        } else if (guiDesign == 3) {
            scoreFont = new Font("MS Sans Serif", Font.BOLD, 14);
        } else {
            scoreFont = new Font("Arial", Font.BOLD, 20);
        }
        
        g.setColor(theme.scoreColor);
        g.setFont(scoreFont);
        g.drawString("Score: " + score, 10, 30);
    }

    private void drawNextPiece(Graphics2D g) {
        Theme theme = currentTheme();
        int previewX = 10;
        int previewY = 100;
        int previewBlockSize = 20;
        int boxSize = 120;

        Font labelFont;
        if (guiDesign == 2) {
            labelFont = new Font("Courier New", Font.BOLD, 14);
        } else if (guiDesign == 3) {
            labelFont = new Font("MS Sans Serif", Font.PLAIN, 11);
        } else {
            labelFont = new Font("Arial", Font.BOLD, 14);
        }

        g.setColor(theme.labelColor);
        g.setFont(labelFont);
        g.drawString("Next:", previewX, previewY);
        g.setColor(theme.boxColor);
        
        if (guiDesign == 3) {
            // Windows 95 beveled box
            g.setColor(theme.background);
            g.fillRect(previewX, previewY + 15, boxSize, boxSize);
            g.setColor(theme.buttonUnselectedLight);
            g.drawLine(previewX, previewY + 15, previewX + boxSize, previewY + 15);
            g.drawLine(previewX, previewY + 15, previewX, previewY + 15 + boxSize);
            g.setColor(theme.buttonUnselectedDark);
            g.drawLine(previewX + boxSize, previewY + 15, previewX + boxSize, previewY + 15 + boxSize);
            g.drawLine(previewX, previewY + 15 + boxSize, previewX + boxSize, previewY + 15 + boxSize);
        } else {
            g.drawRect(previewX, previewY + 15, boxSize, boxSize);
        }

        int[][] nextShape = nextTetromino.shape;
        int shapeWidth = nextShape[0].length;
        int shapeHeight = nextShape.length;
        int offsetX = previewX + (boxSize - shapeWidth * previewBlockSize) / 2;
        int offsetY = previewY + 15 + (boxSize - shapeHeight * previewBlockSize) / 2;

        for (int i = 0; i < nextShape.length; i++) {
            for (int j = 0; j < nextShape[0].length; j++) {
                if (nextShape[i][j] == 1) {
                    int x = offsetX + j * previewBlockSize;
                    int y = offsetY + i * previewBlockSize;
                    Color color = getDesignColor(nextTetromino.color);
                    g.setColor(color);
                    g.fillRect(x, y, previewBlockSize - 1, previewBlockSize - 1);
                    g.setColor(color.darker());
                    g.drawRect(x, y, previewBlockSize - 1, previewBlockSize - 1);
                }
            }
        }
    }

    private void drawHeldPiece(Graphics2D g) {
        Theme theme = currentTheme();
        int previewX = 10;
        int previewY = 350;
        int previewBlockSize = 20;
        int boxSize = 120;

        Font labelFont;
        if (guiDesign == 2) {
            labelFont = new Font("Courier New", Font.BOLD, 14);
        } else if (guiDesign == 3) {
            labelFont = new Font("MS Sans Serif", Font.PLAIN, 11);
        } else {
            labelFont = new Font("Arial", Font.BOLD, 14);
        }

        g.setColor(theme.labelColor);
        g.setFont(labelFont);
        g.drawString("Hold [C]:", previewX, previewY);
        g.setColor(theme.boxColor);
        
        if (guiDesign == 3) {
            // Windows 95 beveled box
            g.setColor(theme.background);
            g.fillRect(previewX, previewY + 15, boxSize, boxSize);
            g.setColor(theme.buttonUnselectedLight);
            g.drawLine(previewX, previewY + 15, previewX + boxSize, previewY + 15);
            g.drawLine(previewX, previewY + 15, previewX, previewY + 15 + boxSize);
            g.setColor(theme.buttonUnselectedDark);
            g.drawLine(previewX + boxSize, previewY + 15, previewX + boxSize, previewY + 15 + boxSize);
            g.drawLine(previewX, previewY + 15 + boxSize, previewX + boxSize, previewY + 15 + boxSize);
        } else {
            g.drawRect(previewX, previewY + 15, boxSize, boxSize);
        }

        if (heldTetromino != null) {
            int[][] holdShape = heldTetromino.shape;
            int shapeWidth = holdShape[0].length;
            int shapeHeight = holdShape.length;
            int offsetX = previewX + (boxSize - shapeWidth * previewBlockSize) / 2;
            int offsetY = previewY + 15 + (boxSize - shapeHeight * previewBlockSize) / 2;

            for (int i = 0; i < holdShape.length; i++) {
                for (int j = 0; j < holdShape[0].length; j++) {
                    if (holdShape[i][j] == 1) {
                        int x = offsetX + j * previewBlockSize;
                        int y = offsetY + i * previewBlockSize;
                        Color color = getDesignColor(heldTetromino.color);
                        g.setColor(color);
                        g.fillRect(x, y, previewBlockSize - 1, previewBlockSize - 1);
                        g.setColor(color.darker());
                        g.drawRect(x, y, previewBlockSize - 1, previewBlockSize - 1);
                    }
                }
            }
        }
    }

    private void drawGameOver(Graphics2D g) {
        switch (guiDesign) {
            case 0:
                drawGameOverDefault(g);
                break;
            case 1:
                drawGameOverSunset(g);
                break;
            case 2:
                drawGameOverRetro(g);
                break;
            case 3:
                drawGameOverWin95(g);
                break;
        }
    }

    private void drawGameOverDefault(Graphics2D g) {
        Theme theme = currentTheme();
        g.setColor(withAlpha(theme.gameOverBackground, 200));
        g.fillRect(0, 0, WIDTH, HEIGHT);
        drawDesignButtons(g);
        g.setColor(theme.gameOverHeaderTextColor);
        g.setFont(new Font("Arial", Font.BOLD, 36));
        g.drawString("GAME OVER", 50, 150);
        g.setFont(new Font("Arial", Font.BOLD, 16));
        g.setColor(theme.gameOverInfoText);
        g.drawString("Final Score: " + score, 70, 200);
        g.drawString("Lines: " + lines + " | Level: " + level, 70, 220);
        g.setColor(theme.gameOverHighScoreLabelColor);
        g.drawString("High Scores:", 70, 250);
        for (int i = 0; i < highScores.size(); i++) {
            g.setColor(i % 2 == 0 ? theme.stat1Color : theme.stat2Color);
            g.drawString((i + 1) + ". " + highScores.get(i), 70, 270 + i * 20);
        }
        g.setColor(theme.gameOverFooterText);
        g.drawString("Press R to Restart | Press D to Change Design", 30, 270 + highScores.size() * 20 + 20);
        
        // Bild für 2 Sekunden anzeigen
        if (gameOverImageTime > 0 && gameOverImage != null) {
            long elapsedTime = System.currentTimeMillis() - gameOverImageTime;
            if (elapsedTime < IMAGE_DISPLAY_TIME) {
                try {
                    int imgWidth = gameOverImage.getWidth();
                    int imgHeight = gameOverImage.getHeight();
                    
                    if (imgWidth > 0 && imgHeight > 0) {
                        int maxWidth = WIDTH - 100;
                        int maxHeight = HEIGHT - 400;
                        double scale = Math.min((double) maxWidth / imgWidth, 
                                               (double) maxHeight / imgHeight);
                        int displayWidth = (int) (imgWidth * scale);
                        int displayHeight = (int) (imgHeight * scale);
                        int x = (WIDTH - displayWidth) / 2;
                        int y = (HEIGHT - displayHeight) / 2;
                        
                        g.drawImage(gameOverImage, x, y, displayWidth, displayHeight, null);
                    }
                } catch (Exception e) {
                    System.err.println("Fehler beim Zeichnen des Bildes: " + e.getMessage());
                    e.printStackTrace();
                }
            }
        }
    }

    private void drawGameOverSunset(Graphics2D g) {
        Theme theme = currentTheme();
        g.setColor(withAlpha(theme.gameOverBackground, 220));
        g.fillRect(0, 0, WIDTH, HEIGHT);
        drawDesignButtons(g);
        
        // Sunset glow effect
        g.setColor(withAlpha(theme.border, 100));
        g.fillRect(40, 130, 300, 80);
        
        g.setColor(theme.gameOverHeaderTextColor);
        g.setFont(new Font("Arial", Font.BOLD, 48));
        g.drawString("GAME OVER", 45, 180);
        
        g.setFont(new Font("Arial", Font.BOLD, 14));
        g.setColor(theme.gameOverInfoText);
        g.drawString("Final Score: " + score, 70, 220);
        g.drawString("Lines: " + lines + " | Level: " + level, 70, 240);
        
        g.setColor(theme.gameOverHighScoreLabelColor);
        g.drawString("Top Scores:", 70, 270);
        for (int i = 0; i < highScores.size(); i++) {
            g.setColor(i % 2 == 0 ? theme.stat1Color : theme.stat2Color);
            g.drawString((i + 1) + ". " + highScores.get(i), 70, 290 + i * 20);
        }
        
        g.setColor(theme.gameOverFooterText);
        g.drawString(">> Press R to Restart | Press D to Change Design <<", 20, 280 + highScores.size() * 20 + 20);
    }

    private void drawGameOverRetro(Graphics2D g) {
        Theme theme = currentTheme();
        g.setColor(withAlpha(Color.DARK_GRAY, 200));
        g.fillRect(0, 0, WIDTH, HEIGHT);
        drawDesignButtons(g);
        
        // Draw retro border
        g.setColor(theme.border);
        g.setStroke(new BasicStroke(4));
        g.drawRect(30, 120, 350, 400);
        
        g.setFont(new Font("Courier New", Font.BOLD, 32));
        g.setColor(theme.gameOverHeaderTextColor);
        g.drawString("GAME OVER", 80, 180);
        
        g.setFont(new Font("Courier New", Font.PLAIN, 14));
        g.setColor(theme.gameOverInfoText);
        g.drawString("Final Score: " + score, 70, 220);
        g.drawString("Lines: " + lines, 70, 240);
        g.drawString("Level: " + level, 70, 260);
        
        g.setColor(theme.gameOverHighScoreLabelColor);
        g.drawString("High Scores:", 70, 290);
        g.setColor(theme.gameOverHighScoreText);
        for (int i = 0; i < highScores.size(); i++) {
            g.drawString((i + 1) + ". " + highScores.get(i), 70, 310 + i * 18);
        }
        
        g.setColor(theme.gameOverFooterText);
        g.drawString("[R] RESTART  [D] DESIGN", 60, 280 + highScores.size() * 18 + 40);
    }

    private void drawGameOverWin95(Graphics2D g) {
        Theme theme = currentTheme();

        // Windows 95 style game over screen
        g.setColor(theme.gameOverBackground);
        g.fillRect(0, 0, WIDTH, HEIGHT);
        
        // Draw window frame
        int winX = 75;
        int winY = 50;
        int winW = 550;
        int winH = 620;
        
        // Window background
        g.setColor(theme.gameOverWindowBackground);
        g.fillRect(winX, winY, winW, winH);
        
        // Outer border (highlight/shadow 3D effect)
        g.setColor(theme.gameOverFrameHighlight);
        g.drawLine(winX, winY, winX + winW, winY);
        g.drawLine(winX, winY, winX, winY + winH);
        
        g.setColor(theme.gameOverFrameShadow);
        g.drawLine(winX + winW, winY, winX + winW, winY + winH);
        g.drawLine(winX, winY + winH, winX + winW, winY + winH);
        
        // Title bar
        g.setColor(theme.gameOverTitleBarColor);
        g.fillRect(winX, winY, winW, 25);
        g.setColor(theme.gameOverTitleTextColor);
        g.setFont(new Font("MS Sans Serif", Font.BOLD, 14));
        g.drawString("Tetris - Game Over", winX + 10, winY + 18);
        
        // Content
        g.setColor(theme.gameOverHeaderTextColor);
        g.setFont(new Font("MS Sans Serif", Font.BOLD, 24));
        g.drawString("GAME OVER", winX + 150, winY + 70);
        
        g.setColor(theme.gameOverInfoText);
        g.setFont(new Font("MS Sans Serif", Font.PLAIN, 12));
        g.drawString("Final Score: " + score, winX + 50, winY + 120);
        g.drawString("Lines Cleared: " + lines, winX + 50, winY + 145);
        g.drawString("Level Reached: " + level, winX + 50, winY + 170);
        
        g.setColor(theme.gameOverHighScoreLabelColor);
        g.drawString("High Scores:", winX + 50, winY + 210);
        
        g.setColor(theme.gameOverHighScoreText);
        for (int i = 0; i < Math.min(5, highScores.size()); i++) {
            g.drawString((i + 1) + ". " + highScores.get(i), winX + 70, winY + 235 + i * 18);
        }
        
        g.setColor(theme.gameOverFooterText);
        g.setFont(new Font("MS Sans Serif", Font.PLAIN, 11));
        g.drawString("Press [R] to Restart", winX + 50, winY + winH - 35);
        
        // Draw buttons below the window
        drawDesignButtons(g);
    }

    private Color getColorForValue(int value) {
        if (value != 0) return Color.WHITE;
        return Color.BLACK;
    }
}

class Tetromino {
    int[][] shape;
    int x;
    int y;
    int color;

    private static final int[][][] SHAPES = {
        {{1, 1, 1, 1}},
        {{1, 1}, {1, 1}},
        {{0, 1, 1}, {1, 1, 0}},
        {{1, 1, 0}, {0, 1, 1}},
        {{1, 0, 0}, {1, 1, 1}},
        {{0, 0, 1}, {1, 1, 1}},
        {{0, 1, 0}, {1, 1, 1}}
    };

    public Tetromino() {
        int randomShape = (int) (Math.random() * SHAPES.length);
        shape = SHAPES[randomShape];
        color = randomShape + 1;
        x = 3;
        y = 0;
    }
}

/**
 * Holds every color used by one visual design ("theme"), loaded from themes.yaml.
 * Optional gameOver* fields fall back to sensible base colors via applyFallbacks()
 * when a theme doesn't define them (see applyFallbacks()).
 */
class Theme {
    String name;
    Color background = Color.BLACK;
    Color grid = new Color(30, 30, 30);
    Color border = Color.WHITE;
    Color scoreColor = Color.WHITE;
    Color labelColor = Color.WHITE;
    Color boxColor = new Color(180, 180, 180);
    Color stat1Color = Color.CYAN;
    Color stat2Color = Color.YELLOW;
    Color stat3Color = Color.MAGENTA;
    List<Color> buttonColors = new ArrayList<>();
    Color buttonTextColor = Color.WHITE;
    Color buttonSelectedLight = Color.WHITE;
    Color buttonSelectedDark = new Color(128, 128, 128);
    Color buttonUnselectedLight = Color.WHITE;
    Color buttonUnselectedDark = Color.BLACK;
    List<Color> paletteColors = new ArrayList<>();

    // Optional, only really used by the Win95 game-over screen; other themes fall back
    // to their base colors for these via applyFallbacks().
    Color gameOverBackground;
    Color gameOverWindowBackground;
    Color gameOverFrameHighlight;
    Color gameOverFrameShadow;
    Color gameOverTitleBarColor;
    Color gameOverTitleTextColor;
    Color gameOverHeaderTextColor;
    Color gameOverInfoText;
    Color gameOverHighScoreLabelColor;
    Color gameOverHighScoreText;
    Color gameOverFooterText;

    /** Fills in any unset optional fields with reasonable derived defaults. Call after parsing. */
    void applyFallbacks() {
        if (gameOverBackground == null) gameOverBackground = background;
        if (gameOverWindowBackground == null) gameOverWindowBackground = background;
        if (gameOverFrameHighlight == null) gameOverFrameHighlight = border;
        if (gameOverFrameShadow == null) gameOverFrameShadow = grid;
        if (gameOverTitleBarColor == null) gameOverTitleBarColor = border;
        if (gameOverTitleTextColor == null) gameOverTitleTextColor = buttonTextColor;
        if (gameOverHeaderTextColor == null) gameOverHeaderTextColor = scoreColor;
        if (gameOverInfoText == null) gameOverInfoText = labelColor;
        if (gameOverHighScoreLabelColor == null) gameOverHighScoreLabelColor = scoreColor;
        if (gameOverHighScoreText == null) gameOverHighScoreText = labelColor;
        if (gameOverFooterText == null) gameOverFooterText = labelColor;
    }

    /** Cycles through paletteColors for tetromino color indices (1-based, wraps around). */
    Color paletteColor(int pieceColorIndex) {
        if (paletteColors.isEmpty()) return Color.GRAY;
        int idx = (pieceColorIndex - 1) % paletteColors.size();
        if (idx < 0) idx += paletteColors.size();
        return paletteColors.get(idx);
    }

    Color buttonColor(int i) {
        if (buttonColors.isEmpty()) return Color.GRAY;
        return buttonColors.get(i % buttonColors.size());
    }
}

/**
 * Minimal, dependency-free YAML reader tailored to the simple structure used by
 * themes.yaml: top-level theme names, 2-space indented "key: value" pairs
 * (value either a scalar or an "[r, g, b]" color), and 2-space indented
 * "key:" headers followed by 4-space indented "- [r, g, b]" list items.
 * This intentionally does not support general YAML syntax.
 */
class ThemeLoader {

    static Map<String, Theme> load(String path) {
        Map<String, Theme> themes = new LinkedHashMap<>();
        try {
            List<String> lines = Files.readAllLines(Paths.get(path));
            Theme current = null;
            String pendingListKey = null;
            List<Color> pendingList = null;

            for (String raw : lines) {
                if (raw.trim().isEmpty() || raw.trim().startsWith("#")) continue;
                int indent = indentOf(raw);
                String line = raw.trim();

                if (indent == 0) {
                    flushList(current, pendingListKey, pendingList);
                    pendingListKey = null;
                    pendingList = null;

                    if (line.endsWith(":")) {
                        String themeName = line.substring(0, line.length() - 1).trim();
                        current = new Theme();
                        current.name = themeName;
                        themes.put(themeName, current);
                    }
                    continue;
                }

                if (current == null) continue;

                if (indent == 2) {
                    flushList(current, pendingListKey, pendingList);
                    pendingListKey = null;
                    pendingList = null;

                    int colon = line.indexOf(':');
                    if (colon == -1) continue;
                    String key = line.substring(0, colon).trim();
                    String value = line.substring(colon + 1).trim();

                    if (value.isEmpty()) {
                        pendingListKey = key;
                        pendingList = new ArrayList<>();
                    } else if (value.startsWith("[")) {
                        assignColor(current, key, parseColor(value));
                    } else {
                        assignScalar(current, key, value);
                    }
                } else if (indent >= 4 && pendingListKey != null && line.startsWith("- ")) {
                    String value = line.substring(2).trim();
                    pendingList.add(parseColor(value));
                }
            }
            flushList(current, pendingListKey, pendingList);

            for (Theme t : themes.values()) {
                t.applyFallbacks();
            }
        } catch (IOException e) {
            System.err.println("Konnte " + path + " nicht laden: " + e.getMessage());
        }
        return themes;
    }

    private static int indentOf(String line) {
        int i = 0;
        while (i < line.length() && line.charAt(i) == ' ') i++;
        return i;
    }

    private static void flushList(Theme theme, String key, List<Color> list) {
        if (theme == null || key == null || list == null) return;
        if (key.equals("buttonColors")) theme.buttonColors = list;
        else if (key.equals("paletteColors")) theme.paletteColors = list;
    }

    private static Color parseColor(String value) {
        String cleaned = value.replace("[", "").replace("]", "").trim();
        String[] parts = cleaned.split(",");
        int r = Integer.parseInt(parts[0].trim());
        int g = Integer.parseInt(parts[1].trim());
        int b = Integer.parseInt(parts[2].trim());
        return new Color(r, g, b);
    }

    private static void assignScalar(Theme t, String key, String value) {
        if (key.equals("name")) t.name = value;
    }

    private static void assignColor(Theme t, String key, Color c) {
        switch (key) {
            case "background": t.background = c; break;
            case "grid": t.grid = c; break;
            case "border": t.border = c; break;
            case "scoreColor": t.scoreColor = c; break;
            case "labelColor": t.labelColor = c; break;
            case "boxColor": t.boxColor = c; break;
            case "stat1Color": t.stat1Color = c; break;
            case "stat2Color": t.stat2Color = c; break;
            case "stat3Color": t.stat3Color = c; break;
            case "buttonTextColor": t.buttonTextColor = c; break;
            case "buttonSelectedLight": t.buttonSelectedLight = c; break;
            case "buttonSelectedDark": t.buttonSelectedDark = c; break;
            case "buttonUnselectedLight": t.buttonUnselectedLight = c; break;
            case "buttonUnselectedDark": t.buttonUnselectedDark = c; break;
            case "gameOverBackground": t.gameOverBackground = c; break;
            case "gameOverWindowBackground": t.gameOverWindowBackground = c; break;
            case "gameOverFrameHighlight": t.gameOverFrameHighlight = c; break;
            case "gameOverFrameShadow": t.gameOverFrameShadow = c; break;
            case "gameOverTitleBarColor": t.gameOverTitleBarColor = c; break;
            case "gameOverTitleTextColor": t.gameOverTitleTextColor = c; break;
            case "gameOverHeaderTextColor": t.gameOverHeaderTextColor = c; break;
            case "gameOverInfoText": t.gameOverInfoText = c; break;
            case "gameOverHighScoreLabelColor": t.gameOverHighScoreLabelColor = c; break;
            case "gameOverHighScoreText": t.gameOverHighScoreText = c; break;
            case "gameOverFooterText": t.gameOverFooterText = c; break;
            default: break;
        }
    }
}
