import java.util.List;
import java.util.ArrayList;

public class BattleArena {

    // PREGUNTA 1: ¿Qué tipo de variable debe ser battleActive y por qué?
    // Se usa `volatile` para asegurar que todos los hilos vean los cambios de esta variable en tiempo real.
    private volatile boolean battleActive = false;

    // Acumulador de daño total causado por todos los guerreros.
    private int totalDamageDealt = 0;

    // Lista compartida de guerreros que participan en la batalla.
    private final List<Warrior> warriors = new ArrayList<>();

    // PREGUNTA 2: Monitores (candados) para sincronización de recursos compartidos.
    // Usamos dos objetos como monitores para sincronización granular:
    private final Object damageLock = new Object();   // Protege el acceso a totalDamageDealt
    private final Object warriorsLock = new Object(); // Protege la lista de guerreros

    public static void main(String[] args) throws InterruptedException {
        BattleArena arena = new BattleArena(); // Crear instancia del campo de batalla
        arena.startBattle();                   // Iniciar la batalla
        Thread.sleep(8000);                    // Esperar 8 segundos para ver el resultado
    }

    // PREGUNTA 3: ¿Debe ser sincronizado este método? ¿Por qué?
    // No es necesario sincronizar el método completo, solo proteger las secciones críticas.
    public void startBattle() {
        battleActive = true;
        System.out.println("⚔️ ¡La batalla ha comenzado!");

        // Crear y registrar 4 guerreros
        for (int i = 0; i < 4; i++) {
            Warrior warrior = new Warrior("Warrior_" + i, this);

            // PREGUNTA 4: Bloque sincronizado para modificar la lista compartida
            synchronized (warriorsLock) {
                warriors.add(warrior); // Acceso seguro a la lista
                System.out.println("🛡️ " + warrior.getName() + " se unió a la batalla");
            }

            // Iniciar el hilo del guerrero que comienza a pelear
            new Thread(warrior::fight, "Fighter_" + i).start();
        }

        // Iniciar el hilo observador que vigilará el estado de la batalla
        new Thread(this::battleObserver, "Observer").start();
    }

    // Método llamado por cada guerrero cuando causa daño
    public void dealDamage(int damage, String attackerName) {
        // PREGUNTA 5: Bloque sincronizado para proteger el recurso totalDamageDealt
        synchronized (damageLock) {
            totalDamageDealt += damage; // Acumular daño
            System.out.println("💥 " + attackerName + " causó " + damage + 
                             " de daño (Total: " + totalDamageDealt + ")");

            // Si el daño supera el umbral, finalizar batalla
            if (totalDamageDealt >= 150) {
                battleActive = false; // Señal para que los guerreros y el observador se detengan
                System.out.println("🏆 ¡Batalla terminada! Daño total: " + totalDamageDealt);

                // Notificar a cualquier hilo que esté esperando el candado damageLock
                damageLock.notifyAll(); // Notifica al observador que debe salir del wait()
            }
        }
    }

    /**
     * Observador que espera notificaciones de que la batalla ha terminado
     */
    private void battleObserver() {
        try {
            synchronized (damageLock) {
                while (battleActive && totalDamageDealt < 150) {
                    System.out.println("👁️ Observador esperando... (Daño actual: " + totalDamageDealt + ")");
                    damageLock.wait(3000); // Espera con timeout (máx 3 segundos)
                }
            }
            System.out.println("👁️ Observador: La batalla ha terminado!");
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt(); // Manejo adecuado de interrupciones
        }
    }

    // Permite que los guerreros sepan si deben seguir peleando
    public boolean isBattleActive() {
        return battleActive; // No necesita sincronización porque es volatile
    }

    // Devuelve la cantidad de guerreros registrados
    public int getWarriorCount() {
        synchronized (warriorsLock) { // Acceso seguro a la lista
            return warriors.size();
        }
    }

    // Devuelve el daño total actual
    public int getTotalDamage() {
        synchronized (damageLock) { // Acceso seguro al contador
            return totalDamageDealt;
        }
    }
}

// Clase que representa un guerrero que participa en la batalla
class Warrior {
    private final String name;          // Nombre del guerrero
    private final BattleArena arena;    // Referencia al campo de batalla
    private int damageDealt = 0;        // Daño causado individualmente

    public Warrior(String name, BattleArena arena) {
        this.name = name;
        this.arena = arena;
    }

    // Método que representa la lógica de ataque del guerrero
    public void fight() {
        while (arena.isBattleActive()) {
            try {
                // Causar daño aleatorio entre 5 y 30
                int damage = (int)(Math.random() * 25) + 5;
                arena.dealDamage(damage, name); // Reportar daño al campo de batalla
                damageDealt += damage;

                // Dormir entre 0.8 y 2 segundos para simular acción
                Thread.sleep(800 + (int)(Math.random() * 1200));
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt(); // Manejo de interrupción
                break;
            }
        }
        System.out.println("⚡ " + name + " terminó la batalla (Daño total: " + damageDealt + ")");
    }

    public String getName() {
        return name;
    }
}
