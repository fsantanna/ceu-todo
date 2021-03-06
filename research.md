# R-0: re-awake check

# R-1: emit self-abortion w/ pointers
        --[[
        -- TODO-RESEARCH-1:
resolvido com proibicao de ponteiros
        -- If an "emit" is on the stack and its enclosing block "self-aborts",
        -- we need to remove the "emit" from the stack because its associated
        -- payload may have gone out of scope:
        --
        --  par/or do
        --      par/or do
        --          var int x;
        --          emit e => &x;
        --      with
        --          await e;    // aborts "emit" block w/ "x"
        --      end
        --  with
        --      px = await e;
        --      *px;            // "x" is out of scope
        --  end
        --
        -- To remove, we need to "finalize" the "emit" removing itself from the
        -- stack:

        --  emit x;
        --      ... becomes ...
        --  do
        --      var int fin = stack_nxti() + sizeof(tceu_stk);
        --                      /* TODO: two levels up */
        --      finalize with
        --          if (fin != CEU_STACK_MAX) then
        --              stack_get(fin)->evt = IN_NONE;
        --          end
        --      end
        --      emit x;
        --      fin = CEU_STACK_MAX;
        --  end
        --
        -- Alternatives:
        --      - forbid pointers in payloads: we could then allow an "emit" to
        --        stay in the stack even if its surrounding block aborts
        --      - statically detect if an "emit" can be aborted
        --          - generate a warning ("slow code") and the code above if it
        --            is the case
        --
        -- TODO: this may have solved the problem with await/awake in the same reaction
        --]]

# R-2: Org termination
        --[[
        -- TODO-RESEARCH-2:
        -- When an organism dies naturally, some pending traversals might
        -- remain in the stack with dangling pointers to the released organism:
        --
        --  class T with
        --  do
        --      par/or do
        --          // aborts and terminates the organism
        --      with
        --          // continuation is cleared but still has to be traversed
        --      end
        --  end
        --
        -- The "stack_clear_org" modifies all pending traversals to the
        -- organism to go in sequence.
        --
        -- Alternatives:
        --      - TODO: currently not organism in sequence, but restart from Main
        --          (#if 0/1 above and in ceu_stack_clear_org)
        --      - ?
        --]]

# R-3: IDs para classes

    u32 id;
        /* TODO: couldn't find a way to use the address as an identifier
         * when killing an organism. The "free" happens before the "kill"
         * completes and the address can be reused in the meantime.
         * It could happen that an await on a newly created organism awakes
         * from the previous "kill".
         * Couldn't reproduce on "tests.lua", but "turtle.ceu" does.
         */
# R-4: mutation during pool iteration
    - solution #1: wacthing + runtime support (it_enter/leave/kill)
    - solution #2: forbig emit/await/spawn/etc

# R-5: emit mais externo vence
    - acho que é o que quero mesmo
        - broadcast é broadcast
        - 2-pass acontece isso
        - 1-pass menor vence

    par/or do
        ...
        emit e => 1;
    with
        await e;
        emit e => 2;
    with
        var int v = await e;
        escape v;   // 1
    end

# R-6: ceu_sys_stack_clear_org
    - Quando um org morre, toda a pilha deve ser atravessada:
        - todos os orgs sendo atravessados que forem filhos do org que morreu, 
          devem ser reapontados para o org seguinte ao que morreu.
    - ANA_NO_NESTED_TERMINATION

# R-7: mutation of root/alias
    Set_pos = '__await',    -- 'adt-mut'
        also for pool references? (& and &&?)

# R-8: org immediate termination
    - interaction with spawn and await
    - SOLUTION: tagged union: [NULL,ALIVE,RET]
        data T with
            tag FAIL;
        or
            tag ALIVE;
        or
            tag TERMINATED with
                var int v;
            end
        end

# R-9: size of trails for internal events double the size of all trails
    - ok_killed and orgs are also expensive

# setjmp & longjmp

We use C calls to start organisms and trails in parallel, awake events, and so 
on.
These calls execute code that might also make new calls, generating a call 
stack during runtime.
In normal situations, after the call completes, the execution resumes to the 
statement following it.
However, due to abortion, some calls in the stack should not resume because its 
body is no longer alive in the program.
For this reason, we use `setjmp` and `longjmp` in the implementation.

Each call that can possibly result in self abortion needs a preceding `setjmp` 
to prepare the stack level to receive a `longjmp`.

The calls are
    - spawning a new org:
        - the new org might emit a global event that awakes a par/or enclosing 
          the call point
    - killing an org:
        - the kill might awake a par/or enclosing the call point
    - mutating an adt:
        - the mutation frees a subtree that might awake a par/or enclosing the 
          call point
    - starting trails in a par:
        - the 1st trail might abort an enclosing par/or, or emit something that 
          does.
    - emit internal event:
        - the emit might awake a par/or enclosing the call point
    - emit external input event:
        - the emit might awake a par/or enclosing the call point
    - traversing children:
        - a child might emit a global event that awakes a par/or enclosing the 
          parent organism with the call point.

The stack level has to hold the current organism and range of trails which the 
call encloses.

Conversely, each abortion point checks the stack to perform the `longjmp`.
The stack is travesed from bottom up to abort the outermost call, making all 
calls above it to never resume.

The `longjmp` sets its continuation in globals so that the resumed call can 
proceed to it, also

with the continuation
