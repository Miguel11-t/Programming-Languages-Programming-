import java.awt.*;
import java.awt.font.FontRenderContext;
import java.awt.geom.Rectangle2D;
import java.awt.image.BufferedImage;
import java.io.BufferedWriter;
import java.io.File;
import java.io.FileWriter;
import java.io.IOException;
import java.util.*;
import javax.imageio.ImageIO;
import java.util.List;
import java.util.Scanner;

public class FenParserApp {
    private static final String AUTOR = "Miguel Àngel Àlvarez";
    private static final Map<Character, String> UNICODE = new HashMap<>();
    static {
        UNICODE.put('K', "♔"); UNICODE.put('Q', "♕"); UNICODE.put('R', "♖"); UNICODE.put('B', "♗"); UNICODE.put('N', "♘"); UNICODE.put('P', "♙");
        UNICODE.put('k', "♚"); UNICODE.put('q', "♛"); UNICODE.put('r', "♜"); UNICODE.put('b', "♝"); UNICODE.put('n', "♞"); UNICODE.put('p', "♟");
    }

    private static void invalid(String msg) {
        System.out.println("invalid fen string");
        System.out.println("error: " + msg);
    }

    public static void main(String[] args) {
        System.out.println("Autor: " + AUTOR);
        String fen = null;
        if (args.length > 0) {
            StringBuilder sb = new StringBuilder();
            for (int i = 0; i < args.length; i++) {
                if (i > 0) sb.append(' ');
                sb.append(args[i]);
            }
            fen = sb.toString();
        } else {
            System.out.println("Una cadena FEN describe una posición de ajedrez en una sola línea de texto, Ejemplo de una cadena FEN: 2r3k1/p3bqp1/Q2p3p/3Pp3/P3N3/8/5PPP/5RK1 b - - 1 27.");
            System.out.print("Cada parte indica,Las piezas en el tablero, A quién le toca mover,  Los enroques posibles, Casilla en passant,  Medio movimiento, Número de movimiento completo.");
            System.out.println("___________________________________________");
            System.out.println("///////Ahora copia la cadena FEN////////");

            try (Scanner sc = new Scanner(System.in)) {
                if (sc.hasNextLine()) fen = sc.nextLine();
                else { System.out.println("nada ingresado. saliendo."); return; }
            }

        }

        fen = fen.trim();
        ValidationResult res = validarFen(fen);
        if (!res.ok) { invalid(res.error); return; }

        System.out.println("fen válida! tablero:");
        mostrarTablero(res.board);
        System.out.println("turno: " + res.turn + "  enroque: " + res.castling + "  eps: " + res.eps +
                "  halfmove: " + res.halfmove + "  fullmove: " + res.fullmove);

        try {
            String out = "board.png";
            generarImagen(res.board, out);
            System.out.println("imagen generada: " + out);
        } catch (Exception e) {
            System.out.println("no se pudo generar la imagen: " + e.getMessage());
        }

        try { crearReadme(); System.out.println("README.md creado."); } catch (IOException ex) { }
    }

    private static class ValidationResult {
        boolean ok;
        String error;
        List<char[]> board;
        String turn, castling, eps;
        int halfmove, fullmove;
    }

    public static ValidationResult validarFen(String fen) {
        ValidationResult r = new ValidationResult();
        r.ok = false;
        if (fen == null || fen.isEmpty()) { r.error = "cadena vacía"; return r; }

        String[] parts = fen.split("\\s+");
        if (parts.length != 6) { r.error = "deben ser 6 campos separados por espacios (pieza turno enroque eps reloj movimientos). encontré: " + parts.length; return r; }
        String pieces = parts[0];
        String turn = parts[1];
        String castling = parts[2];
        String eps = parts[3];
        String half = parts[4];
        String full = parts[5];

        String[] ranks = pieces.split("/");
        if (ranks.length != 8) { r.error = "la parte de piezas debe tener 8 filas separadas por '/'. encontré: " + ranks.length; return r; }
        List<char[]> board = new ArrayList<>();

        for (int i = 0; i < 8; i++) {
            String rank = ranks[i];
            int count = 0;
            List<Character> row = new ArrayList<>();
            for (int j = 0; j < rank.length(); j++) {
                char ch = rank.charAt(j);
                if (Character.isDigit(ch)) {
                    if (ch == '0') { r.error = "0 no permitido en fila " + (8 - i); return r; }
                    if (ch < '1' || ch > '8') { r.error = "número inválido '"+ch+"' en fila " + (8 - i); return r; }
                    int n = ch - '0';
                    count += n;
                    for (int k = 0; k < n; k++) row.add('.');
                } else {
                    if ("PNBRQKpnbrqk".indexOf(ch) == -1) { r.error = "carácter inválido '"+ch+"' en fila " + (8 - i); return r; }
                    count += 1; row.add(ch);
                }
                if (count > 8) { r.error = "la fila " + (8 - i) + " tiene más de 8 casillas."; return r; }
            }
            if (count < 8) { r.error = "la fila " + (8 - i) + " tiene menos de 8 casillas."; return r; }
            char[] arr = new char[8];
            for (int p = 0; p < 8; p++) arr[p] = row.get(p);
            board.add(arr);
        }

        if (!turn.equals("w") && !turn.equals("b")) { r.error = "turno inválido: debe ser 'w' o 'b'."; return r; }

        if (!castling.equals("-")) {
            if (!castling.matches("[KQkq]{1,4}")) { r.error = "campo de enroque inválido: debe ser '-' o combinación de KQkq sin otros símbolos."; return r; }
            for (char c : castling.toCharArray()) if (countOccurrences(castling, c) > 1) { r.error = "letra repetida en enroque: '"+c+"'."; return r; }
        }

        if (!eps.equals("-")) {
            if (!eps.matches("[a-h][36]")) { r.error = "casilla en passant inválida: debe ser '-' o una casilla como 'e3' o 'd6'."; return r; }
        }

        if (!half.matches("\\d+")) { r.error = "reloj de medio movimiento inválido: debe ser entero no negativo."; return r; }

        if (!full.matches("[1-9]\\d*")) { r.error = "contador de movimientos inválido: debe ser entero >= 1."; return r; }

        int kWhite = 0, kBlack = 0;
        for (char[] row : board) for (char c : row) { if (c == 'K') kWhite++; if (c == 'k') kBlack++; }
        if (kWhite != 1) { r.error = "debe haber exactamente 1 rey blanco (K). encontré: " + kWhite; return r; }
        if (kBlack != 1) { r.error = "debe haber exactamente 1 rey negro (k). encontré: " + kBlack; return r; }

        r.ok = true;
        r.board = board;
        r.turn = turn; r.castling = castling; r.eps = eps; r.halfmove = Integer.parseInt(half); r.fullmove = Integer.parseInt(full);
        return r;
    }

    private static int countOccurrences(String s, char c) { int cnt=0; for (char x: s.toCharArray()) if (x==c) cnt++; return cnt; }

    public static void mostrarTablero(List<char[]> board) {
        System.out.println("   a b c d e f g h");
        System.out.println("  +----------------+");
        int rank = 8;
        for (char[] row : board) {
            StringBuilder sb = new StringBuilder();
            sb.append(rank).append(" |");
            for (int i = 0; i < 8; i++) {
                char c = row[i];
                if (c == '.') sb.append(" .");
                else {
                    String glyph = UNICODE.get(c);
                    if (glyph == null) glyph = String.valueOf(c);
                    sb.append(' ').append(glyph);
                }
            }
            sb.append(" |");
            System.out.println(sb.toString());
            rank--;
        }
        System.out.println("  +----------------+");
    }

    public static void generarImagen(List<char[]> board, String outFile) throws IOException {
        final int square = 64;
        final int size = square * 8;
        BufferedImage img = new BufferedImage(size, size, BufferedImage.TYPE_INT_ARGB);
        Graphics2D g = img.createGraphics();
        g.setRenderingHint(RenderingHints.KEY_ANTIALIASING, RenderingHints.VALUE_ANTIALIAS_ON);
        g.setRenderingHint(RenderingHints.KEY_TEXT_ANTIALIASING, RenderingHints.VALUE_TEXT_ANTIALIAS_ON);

        Color light = new Color(240, 217, 181);
        Color dark  = new Color(181, 136, 99);
        for (int r = 0; r < 8; r++) {
            for (int f = 0; f < 8; f++) {
                int x = f * square;
                int y = r * square;
                boolean isLight = ((r + f) % 2 == 0);
                g.setColor(isLight ? light : dark);
                g.fillRect(x, y, square, square);
            }
        }

        g.setColor(Color.BLACK);
        Font coordFont = new Font("SansSerif", Font.PLAIN, 14);
        g.setFont(coordFont);
        for (int i = 0; i < 8; i++) {
            String file = String.valueOf((char)('a' + i));
            int x = i * square + 6;
            int y = 8 * square - 6;
            g.drawString(file, x, y);
        }

        Font pieceFont = new Font("Serif", Font.PLAIN, 40);
        g.setFont(pieceFont);
        FontRenderContext frc = g.getFontRenderContext();

        for (int r = 0; r < 8; r++) {
            for (int f = 0; f < 8; f++) {
                char c = board.get(r)[f];
                if (c == '.') continue;
                String glyph = UNICODE.get(c);
                String draw = (glyph != null) ? glyph : String.valueOf(c);
                boolean isWhitePiece = Character.isUpperCase(c);
                Color textColor = isWhitePiece ? Color.BLACK : Color.WHITE;
                int x = f * square;
                int y = r * square;
                Rectangle2D bounds = pieceFont.getStringBounds(draw, frc);
                int tw = (int) bounds.getWidth();
                int th = (int) bounds.getHeight();
                int tx = x + (square - tw) / 2;
                int ty = y + (square + th) / 2 - 6;

                if (!isWhitePiece) {
                    g.setColor(new Color(0,0,0,120));
                    g.drawString(draw, tx+1, ty+1);
                }
                g.setColor(textColor);
                g.drawString(draw, tx, ty);
            }
        }

        g.dispose();
        ImageIO.write(img, "png", new File(outFile));
    }

    private static void crearReadme() throws IOException {
        String txt = "# parser-fen-java\n\n" +
                "proyecto: parser para cadenas FEN (forsyth-edwards notation).\n\n" +
                "esto es una versión lista para entregar: valida la fen, imprime el tablero en consola y genera un archivo board.png con la posición.\n\n" +
                "## archivos\n" +
                "- FenParserApp.java  (código fuente)\n" +
                "- board.png (generado al ejecutar con una fen válida)\n\n" +
                "## cómo compilar\n" +
                "````bash\n" +
                "javac FenParserApp.java\n" +
                "````\n\n" +
                "## cómo ejecutar\n" +
                "modo interactivo:\n" +
                "````bash\n" +
                "java FenParserApp\n" +
                "````\n\n" +
                "modo no interactivo (pasa la fen entre comillas):\n" +
                "````bash\n" +
                "java FenParserApp \"2r3k1/p3bqp1/Q2p3p/3Pp3/P3N3/8/5PPP/5RK1 b - - 1 27\"\n" +
                "````\n\n" +
                "## notas\n" +
                "- los mensajes están en español y en minúsculas (estilo informal).\n" +
                "- si querés usar imágenes reales de piezas, podés modificar el método generarImagen para cargar pngs desde resources/ y dibujarlos en cada casilla.\n" +
                "- el programa hace validaciones extra (por ejemplo, que haya exactamente 1 rey blanco y 1 rey negro).\n\n" +
                "## autores\n" +
                "- Miguel Orozco\n";
        try (BufferedWriter w = new BufferedWriter(new FileWriter("README.md"))) { w.write(txt); }
    }
}
