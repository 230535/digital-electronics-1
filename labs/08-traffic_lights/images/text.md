library IEEE;
use IEEE.STD_LOGIC_1164.ALL;

-- Uncomment the following library declaration if using
-- arithmetic functions with Signed or Unsigned values
--use IEEE.NUMERIC_STD.ALL;

-- Uncomment the following library declaration if instantiating
-- any Xilinx leaf cells in this code.
--library UNISIM;
--use UNISIM.VComponents.all;

entity top is
    Port ( CLK100MHZ : in STD_LOGIC;
           LED16_R : out STD_LOGIC;
           LED16_G : out STD_LOGIC;
           LED16_B : out STD_LOGIC;
           LED17_R : out STD_LOGIC;
           LED17_G : out STD_LOGIC;
           LED17_B : out STD_LOGIC;
           BTNC : in STD_LOGIC);
end top;

architecture Behavioral of top is

begin
    --------------------------------------------------------
    -- Instance (copy) of tlc entity
    tlc : entity work.tlc
        port map(
            clk   => CLK100MHZ,
            reset => BTNC,
            south_o(2)=> LED16_R,
            south_o(1)=> LED16_G,
            south_o(0)=> LED16_B,
            west_o(2)=> LED17_R,
            west_o(1)=> LED17_G,
            west_o(0)=> LED17_B        
            -- MAP TWO RGB LEDS HERE

        );

end architecture Behavioral;













------------------------------------------------------------
--
-- Traffic lights controller testbench.
-- Nexys A7-50T, Vivado v2020.1.1, EDA Playground
--
-- Copyright (c) 2020-Present Tomas Fryza
-- Dept. of Radio Electronics, Brno Univ. of Technology, Czechia
-- This work is licensed under the terms of the MIT license.
--
------------------------------------------------------------

library ieee;
use ieee.std_logic_1164.all;

------------------------------------------------------------
-- Entity declaration for testbench
------------------------------------------------------------
entity tb_tlc is
    -- Entity of testbench is always empty
end entity tb_tlc;

------------------------------------------------------------
-- Architecture body for testbench
------------------------------------------------------------
architecture testbench of tb_tlc is

    -- Local constants
    constant c_CLK_100MHZ_PERIOD : time := 10 ns;

    --Local signals
    signal s_clk_100MHz : std_logic;
    signal s_reset      : std_logic;
    signal s_south      : std_logic_vector(3 - 1 downto 0);
    signal s_west       : std_logic_vector(3 - 1 downto 0);

begin
    -- Connecting testbench signals with tlc entity
    -- (Unit Under Test)
    uut_tlc : entity work.tlc
        port map(
            clk     => s_clk_100MHz,
            reset   => s_reset,
            south_o => s_south,
            west_o  => s_west
        );

    --------------------------------------------------------
    -- Clock generation process
    --------------------------------------------------------
    p_clk_gen : process
    begin
        while now < 10000 ns loop -- 10 usec of simulation
            s_clk_100MHz <= '0';
            wait for c_CLK_100MHZ_PERIOD / 2;
            s_clk_100MHz <= '1';
            wait for c_CLK_100MHZ_PERIOD / 2;
        end loop;
        wait;
    end process p_clk_gen;

    --------------------------------------------------------
    -- Reset generation process
    --------------------------------------------------------
    p_reset_gen : process
    begin
        s_reset <= '0'; wait for 200 ns;
        -- Reset activated
        s_reset <= '1'; wait for 500 ns;
        -- Reset deactivated
        s_reset <= '0';
        wait;
    end process p_reset_gen;

    --------------------------------------------------------
    -- Data generation process
    --------------------------------------------------------
    p_stimulus : process
    begin
        -- No input data needed.
        wait;
    end process p_stimulus;

end architecture testbench;
























------------------------------------------------------------
--
-- Traffic light controller using FSM.
-- Nexys A7-50T, Vivado v2020.1.1, EDA Playground
--
-- Copyright (c) 2020-Present Tomas Fryza
-- Dept. of Radio Electronics, Brno Univ. of Technology, Czechia
-- This work is licensed under the terms of the MIT license.
--
-- This code is inspired by:
-- [1] LBEbooks, Lesson 92 - Example 62: Traffic Light Controller
--     https://www.youtube.com/watch?v=6_Rotnw1hFM
-- [2] David Williams, Implementing a Finite State Machine in VHDL
--     https://www.allaboutcircuits.com/technical-articles/implementing-a-finite-state-machine-in-vhdl/
-- [3] VHDLwhiz, One-process vs two-process vs three-process state machine
--     https://vhdlwhiz.com/n-process-state-machine/
--
------------------------------------------------------------

library ieee;
use ieee.std_logic_1164.all;
use ieee.numeric_std.all;

------------------------------------------------------------
-- Entity declaration for traffic light controller
------------------------------------------------------------
entity tlc is
    port(
        clk     : in  std_logic;
        reset   : in  std_logic;
        -- Traffic lights (RGB LEDs) for two directions
        south_o : out std_logic_vector(3 - 1 downto 0);
        west_o  : out std_logic_vector(3 - 1 downto 0)
    );
end entity tlc;

------------------------------------------------------------
-- Architecture declaration for traffic light controller
------------------------------------------------------------
architecture Behavioral of tlc is

    -- Define the states
    type t_state is (STOP1,
                     WEST_GO,
                     WEST_WAIT,
                     STOP2,
                     SOUTH_GO,
                     SOUTH_WAIT);
    -- Define the signal that uses different states
    signal s_state : t_state;

    -- Internal clock enable
    signal s_en : std_logic;

    -- Local delay counter
    signal s_cnt : unsigned(4 downto 0);

    -- Specific values for local counter
    constant c_DELAY_4SEC : unsigned(4 downto 0) := b"1_0000";
    constant c_DELAY_2SEC : unsigned(4 downto 0) := b"0_1000";
    constant c_DELAY_1SEC : unsigned(4 downto 0) := b"0_0100";
    constant c_ZERO       : unsigned(4 downto 0) := b"0_0000";

    -- Output values
    constant c_RED        : std_logic_vector(2 downto 0) := b"100";
    constant c_YELLOW     : std_logic_vector(2 downto 0) := b"110";
    constant c_GREEN      : std_logic_vector(2 downto 0) := b"010";

begin

    --------------------------------------------------------
    -- Instance (copy) of clock_enable entity generates 
    -- an enable pulse every 250 ms (4 Hz). Remember that 
    -- the frequency of the clock signal is 100 MHz.
    clk_en0 : entity work.clock_enable
        generic map(
            g_MAX => 1 -- FOR SIMULATION PURPOSE ONLY !!!
            -- FOR IMPLEMENTATION: g_MAX = 250 ms / (1/100 MHz)
        )
        port map(
            clk   => clk,
            reset => reset,
            ce_o  => s_en
        );

    --------------------------------------------------------
    -- p_traffic_fsm:
    -- The sequential process with synchronous reset and 
    -- clock_enable entirely controls the s_state signal by 
    -- CASE statement.
    --------------------------------------------------------
    p_traffic_fsm : process(clk)
    begin
        if rising_edge(clk) then
            if (reset = '1') then   -- Synchronous reset
                s_state <= STOP1;   -- Set initial state
                s_cnt   <= c_ZERO;  -- Clear delay counter

            elsif (s_en = '1') then
                -- Every 250 ms, CASE checks the value of the s_state 
                -- variable and changes to the next state according 
                -- to the delay value.
                case s_state is

                    -- If the current state is STOP1, then wait 1 sec
                    -- and move to the next GO_WAIT state.
                    when STOP1 =>
                        -- Count up to c_DELAY_1SEC
                        if (s_cnt < c_DELAY_1SEC) then
                            s_cnt <= s_cnt + 1;
                        else
                            -- Move to the next state
                            s_state <= WEST_GO;
                            -- Reset local counter value
                            s_cnt <= c_ZERO;
                        end if;

                    when WEST_GO =>
                    -- Count up to c_DELAY_1SEC
                        if (s_cnt < c_DELAY_4SEC) then
                            s_cnt <= s_cnt + 1;
                        else
                            -- Move to the next state
                            s_state <= WEST_WAIT;
                            -- Reset local counter value
                            s_cnt <= c_ZERO;
                        end if;
                        -- WRITE OTHER STATES HERE
                    when WEST_WAIT =>
                    -- Count up to c_DELAY_1SEC
                        if (s_cnt < c_DELAY_2SEC) then
                            s_cnt <= s_cnt + 1;
                        else
                            -- Move to the next state
                            s_state <= STOP2;
                            -- Reset local counter value
                            s_cnt <= c_ZERO;
                        end if;
                    when STOP2 =>
                    -- Count up to c_DELAY_1SEC
                        if (s_cnt < c_DELAY_1SEC) then
                            s_cnt <= s_cnt + 1;
                        else
                            -- Move to the next state
                            s_state <= SOUTH_GO;
                            -- Reset local counter value
                            s_cnt <= c_ZERO;
                        end if;
                    when SOUTH_GO =>
                    -- Count up to c_DELAY_1SEC
                        if (s_cnt < c_DELAY_4SEC) then
                            s_cnt <= s_cnt + 1;
                        else
                            -- Move to the next state
                            s_state <= SOUTH_WAIT;
                            -- Reset local counter value
                            s_cnt <= c_ZERO;
                        end if;                                                
                    when SOUTH_WAIT =>
                    -- Count up to c_DELAY_1SEC
                        if (s_cnt < c_DELAY_2SEC) then
                            s_cnt <= s_cnt + 1;
                        else
                            -- Move to the next state
                            s_state <= STOP1;
                            -- Reset local counter value
                            s_cnt <= c_ZERO;
                        end if;
                    -- It is a good programming practice to use the 
                    -- OTHERS clause, even if all CASE choices have 
                    -- been made.
                    when others =>
                        s_state <= STOP1;
                        s_cnt   <= c_ZERO;
                end case;
            end if; -- Synchronous reset
        end if; -- Rising edge
    end process p_traffic_fsm;

    --------------------------------------------------------
    -- p_output_fsm:
    -- The combinatorial process is sensitive to state
    -- changes and sets the output signals accordingly.
    -- This is an example of a Moore state machine and
    -- therefore the output is set based on the active state.
    --------------------------------------------------------
    p_output_fsm : process(s_state)
    begin
        case s_state is
            when STOP1 =>
                south_o <= c_RED;
                west_o  <= c_RED;
            when WEST_GO =>
                south_o <= c_RED;
                west_o  <= c_GREEN;          
                -- WRITE OTHER STATES HERE
            when WEST_WAIT =>
                south_o <= c_RED;
                west_o  <= c_YELLOW;
            when STOP2 =>
                south_o <= c_RED;
                west_o  <= c_RED;
            when SOUTH_GO =>
                south_o <= c_GREEN;
                west_o  <= c_RED;
            when SOUTH_WAIT =>
                south_o <= c_YELLOW;
                west_o  <= c_RED;                               
            when others =>
                south_o <= c_RED;
                west_o  <= c_RED;
        end case;
    end process p_output_fsm;

end architecture Behavioral;
