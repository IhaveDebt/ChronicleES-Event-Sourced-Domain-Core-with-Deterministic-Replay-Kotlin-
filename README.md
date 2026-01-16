import java.time.Instant
import java.util.UUID

// ===== Domain Events =====
sealed class Event {
    abstract val id: UUID
    abstract val timestamp: Instant
}

data class AccountCreated(
    override val id: UUID,
    override val timestamp: Instant,
    val owner: String
) : Event()

data class FundsDeposited(
    override val id: UUID,
    override val timestamp: Instant,
    val amount: Long
) : Event()

data class FundsWithdrawn(
    override val id: UUID,
    override val timestamp: Instant,
    val amount: Long
) : Event()

// ===== Aggregate State =====
data class AccountState(
    val owner: String,
    val balance: Long,
    val active: Boolean
) {
    companion object {
        fun empty() = AccountState("", 0, false)
    }
}

// ===== Event Application =====
fun applyEvent(state: AccountState, event: Event): AccountState =
    when (event) {
        is AccountCreated ->
            if (state.active) error("Account already exists")
            else AccountState(event.owner, 0, true)

        is FundsDeposited ->
            if (!state.active) error("Account inactive")
            else state.copy(balance = state.balance + event.amount)

        is FundsWithdrawn ->
            if (!state.active) error("Account inactive")
            else if (state.balance < event.amount) error("Insufficient funds")
            else state.copy(balance = state.balance - event.amount)
    }

// ===== Event Store =====
class EventStore {
    private val events = mutableListOf<Event>()

    fun append(event: Event) {
        events.add(event)
    }

    fun all(): List<Event> = events.toList()
}

// ===== Aggregate Root =====
class Account(private val store: EventStore) {
    private var state = AccountState.empty()

    fun replay() {
        state = AccountState.empty()
        store.all().forEach {
            state = applyEvent(state, it)
        }
    }

    fun create(owner: String) {
        record(AccountCreated(UUID.randomUUID(), Instant.now(), owner))
    }

    fun deposit(amount: Long) {
        record(FundsDeposited(UUID.randomUUID(), Instant.now(), amount))
    }

    fun withdraw(amount: Long) {
        record(FundsWithdrawn(UUID.randomUUID(), Instant.now(), amount))
    }

    private fun record(event: Event) {
        state = applyEvent(state, event)
        store.append(event)
    }

    fun snapshot(): AccountState = state
}

// ===== Audit =====
fun audit(events: List<Event>) {
    println("\n=== Event Audit Log ===")
    events.forEach {
        println("${it.timestamp} -> ${it::class.simpleName}")
    }
}

// ===== Demo =====
fun main() {
    val store = EventStore()
    val account = Account(store)

    account.create("Alice")
    account.deposit(500)
    account.withdraw(120)

    println("Current State:")
    println(account.snapshot())

    audit(store.all())

    println("\nReplaying from scratch...")
    account.replay()
    println("Replayed State:")
    println(account.snapshot())
}
