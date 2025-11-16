<xml xmlns="http://www.w3.org/1999/xhtml" is_dbot="true" collection="false">
  <variables>
    <variable id="mart:result" islocal="false">martingale:resultIsWin</variable>
    <variable id="mart:profit" islocal="false">martingale:profit</variable>
    <variable id="mart:total" islocal="false">martingale:totalProfit</variable>
    <variable id="mart:again" islocal="false">martingale:tradeAgain</variable>
    <variable id="mart:profitTh" islocal="false">martingale:profitThreshold</variable>
    <variable id="mart:size" islocal="false">martingale:size</variable>
    <variable id="mart:init" islocal="false">martingale:initialStake</variable>
    <variable id="mart:lossTh" islocal="false">martingale:lossThreshold</variable>
    <variable id="mart:won" islocal="false">won</variable>
    <variable id="notify:total" islocal="false">Notification:totalProfit</variable>
    <variable id="notify:loss" islocal="false">Notification:lossThresholdReached</variable>
    <variable id="notify:profit" islocal="false">Notification:profitThresholdReached</variable>
    <variable id="mart:mult" islocal="false">martingale:multiplier</variable>
    <variable id="maxStakeVar" islocal="false">maxStake</variable>
    <variable id="useMax" islocal="false">setMaxStake?</variable>
  </variables>

  <!-- Trade parameters: Synthetic volatility index 1s, Vol100 (symbol 1HZ100V), Digit Under (prediction 6), duration 1 tick -->
  <block type="trade_definition" id="trade_def" x="0" y="0">
    <statement name="TRADE_OPTIONS">
      <block type="trade_definition_market" id="market_block">
        <field name="MARKET_LIST">synthetic_index</field>
        <field name="SUBMARKET_LIST">random_index</field>
        <field name="SYMBOL_LIST">1HZ100V</field>
        <next>
          <block type="trade_definition_tradetype" id="tradetype_block">
            <field name="TRADETYPECAT_LIST">digits</field>
            <field name="TRADETYPE_LIST">overunder</field>
            <next>
              <block type="trade_definition_contracttype" id="contract_block">
                <field name="TYPE_LIST">DIGITUNDER</field>
                <next>
                  <block type="trade_definition_candleinterval" id="candle_block">
                    <field name="CANDLEINTERVAL_LIST">60</field>
                    <next>
                      <block type="trade_definition_tradeoptions" id="tradeopts">
                        <mutation has_first_barrier="false" has_second_barrier="false" has_prediction="true"></mutation>
                        <field name="DURATIONTYPE_LIST">t</field>
                        <value name="DURATION">
                          <shadow type="math_number"><field name="NUM">1</field></shadow>
                        </value>
                        <value name="AMOUNT">
                          <block type="procedures_callreturn" id="call_amount">
                            <mutation name="Martingale Trade Amount"></mutation>
                          </block>
                        </value>
                        <value name="PREDICTION">
                          <shadow type="math_number_positive"><field name="NUM">6</field></shadow>
                        </value>
                      </block>
                    </next>
                  </block>
                </next>
              </block>
            </next>
          </block>
        </next>
      </block>
    </statement>

    <!-- Initialization: default parameter values and safety caps -->
    <statement name="INITIALIZATION">
      <block type="variables_set" id="init_useMax">
        <field name="VAR">setMaxStake?</field>
        <value name="VALUE"><block type="logic_boolean"><field name="BOOL">TRUE</field></block></value>
        <next>
          <block type="variables_set" id="init_initialStake">
            <field name="VAR">martingale:initialStake</field>
            <value name="VALUE"><block type="math_number"><field name="NUM">0.35</field></block></value>
            <next>
              <block type="variables_set" id="init_maxStake">
                <field name="VAR">maxStake</field>
                <value name="VALUE"><block type="math_number"><field name="NUM">10</field></block></value>
                <next>
                  <block type="variables_set" id="init_profitTh">
                    <field name="VAR">martingale:profitThreshold</field>
                    <value name="VALUE"><block type="math_number"><field name="NUM">20</field></block></value>
                    <next>
                      <block type="variables_set" id="init_lossTh">
                        <field name="VAR">martingale:lossThreshold</field>
                        <value name="VALUE"><block type="math_number"><field name="NUM">100</field></block></value>
                        <next>
                          <block type="variables_set" id="init_size">
                            <field name="VAR">martingale:size</field>
                            <value name="VALUE"><block type="math_number"><field name="NUM">2</field></block></value>
                            <next>
                              <block type="variables_set" id="init_mult">
                                <field name="VAR">martingale:multiplier</field>
                                <value name="VALUE"><block type="math_number"><field name="NUM">1</field></block></value>
                              </block>
                            </next>
                          </block>
                        </next>
                      </block>
                    </next>
                  </block>
                </next>
              </block>
            </next>
          </block>
        </next>
      </block>
    </statement>
  </block>

  <!-- Before purchase: purchase block uses DIGITUNDER as defined -->
  <block type="before_purchase" id="before_purchase" x="0" y="500">
    <statement name="BEFOREPURCHASE_STACK">
      <block type="purchase" id="purchase_block">
        <field name="PURCHASE_LIST">DIGITUNDER</field>
      </block>
    </statement>
  </block>

  <!-- After purchase: update totals and decide whether to restart martingale or reset -->
  <block type="after_purchase" id="after_purchase" x="600" y="0">
    <statement name="AFTERPURCHASE_STACK">
      <block type="controls_if" id="after_if">
        <value name="IF0">
          <!-- read contract result: pass profit (index 4) and win/lose status -->
          <block type="procedures_callreturn" id="call_mart_again">
            <mutation name="Martingale Trade Again After Purchase">
              <arg name="martingale:profit"></arg>
              <arg name="martingale:resultIsWin"></arg>
            </mutation>
            <data>mart_again_id</data>
            <value name="ARG0">
              <block type="read_details"><field name="DETAIL_INDEX">4</field></block>
            </value>
            <value name="ARG1">
              <block type="contract_check_result"><field name="CHECK_RESULT">win</field></block>
            </value>
          </block>
        </value>
        <statement name="DO0">
          <block type="trade_again"></block>
        </statement>
      </block>
    </statement>
  </block>

  <!-- Procedure: Martingale Trade Amount (returns stake = min(multiplier*initialStake, maxStake)) -->
  <block type="procedures_defreturn" id="proc_amount" x="0" y="800">
    <field name="NAME">Martingale Trade Amount</field>
    <statement name="STACK">
      <block type="variables_set" id="calc_stake">
        <field name="VAR">martingale:tradeAgain</field>
        <value name="VALUE">
          <!-- temporary: we'll compute stake then return as value -->
          <block type="math_arithmetic">
            <field name="OP">MULTIPLY</field>
            <value name="A"><block type="variables_get"><field name="VAR">martingale:multiplier</field></block></value>
            <value name="B"><block type="variables_get"><field name="VAR">martingale:initialStake</field></block></value>
          </block>
        </value>
        <next>
          <block type="controls_if" id="cap_if">
            <value name="IF0">
              <block type="logic_operation"><field name="OP">AND</field>
                <value name="A"><block type="variables_get"><field name="VAR">setMaxStake?</field></block></value>
                <value name="B">
                  <block type="logic_compare"><field name="OP">GT</field>
                    <value name="A">
                      <block type="procedures_callreturn"><mutation name="GetCurrentStakeTemp"></mutation></block>
                    </value>
                    <value name="B"><block type="variables_get"><field name="VAR">maxStake</field></block></value>
                  </block>
                </value>
              </block>
            </value>
            <statement name="DO0">
              <!-- if exceeds max, set to max -->
              <block type="variables_set"><field name="VAR">martingale:tradeAgain</field>
                <value name="VALUE"><block type="variables_get"><field name="VAR">maxStake</field></block></value>
              </block>
            </statement>
          </block>
        </next>
      </block>
    </statement>
    <value name="RETURN">
      <block type="variables_get"><field name="VAR">martingale:tradeAgain</field></block>
    </value>
  </block>

  <!-- Procedure: Martingale Trade Again After Purchase -->
  <block type="procedures_defreturn" id="proc_after" x="300" y="800">
    <mutation>
      <arg name="martingale:profit"></arg>
      <arg name="martingale:resultIsWin"></arg>
    </mutation>
    <field name="NAME">Martingale Trade Again After Purchase</field>
    <statement name="STACK">
      <!-- Add profit of last trade to totalProfit -->
      <block type="math_change">
        <field name="VAR">martingale:totalProfit</field>
        <value name="DELTA"><block type="variables_get"><field name="VAR">martingale:profit</field></block></value>
        <next>
          <!-- Reset or increase multiplier based on last result -->
          <block type="controls_if">
            <value name="IF0"><block type="variables_get"><field name="VAR">martingale:resultIsWin</field></block></value>
            <statement name="DO0">
              <!-- if win -> reset multiplier to 1 -->
              <block type="variables_set"><field name="VAR">martingale:multiplier</field>
                <value name="VALUE"><block type="math_number"><field name="NUM">1</field></block></value>
              </block>
            </statement>
            <statement name="ELSE">
              <!-- if loss -> multiply multiplier by martingale:size -->
              <block type="variables_set"><field name="VAR">martingale:multiplier</field>
                <value name="VALUE">
                  <block type="math_arithmetic"><field name="OP">MULTIPLY</field>
                    <value name="A"><block type="variables_get"><field name="VAR">martingale:multiplier</field></block></value>
                    <value name="B"><block type="variables_get"><field name="VAR">martingale:size</field></block></value>
                  </block>
                </value>
              </block>
            </statement>
            <next>
              <!-- Notifications and stop conditions -->
              <block type="controls_if">
                <value name="IF0">
                  <block type="logic_compare"><field name="OP">LT</field>
                    <value name="A"><block type="variables_get"><field name="VAR">martingale:totalProfit</field></block></value>
                    <value name="B"><block type="variables_get"><field name="VAR">martingale:profitThreshold</field></block></value>
                  </block>
                </value>
                <statement name="DO0">
                  <!-- still below profit target: check loss threshold -->
                  <block type="controls_if">
                    <value name="IF0">
                      <block type="logic_compare"><field name="OP">GT</field>
                        <value name="A"><block type="variables_get"><field name="VAR">martingale:totalProfit</field></block></value>
                        <value name="B">
                          <block type="math_single"><field name="OP">NEG</field>
                            <value name="NUM"><block type="variables_get"><field name="VAR">martingale:lossThreshold</field></block></value>
                          </block>
                        </value>
                      </block>
                    </value>
                    <statement name="DO0">
                      <!-- within loss threshold -> continue martingale -->
                      <block type="variables_set"><field name="VAR">martingale:tradeAgain</field>
                        <value name="VALUE"><block type="logic_boolean"><field name="BOOL">TRUE</field></block></value>
                      </block>
                    </statement>
                    <statement name="ELSE">
                      <!-- loss threshold exceeded -> stop -->
                      <block type="variables_set"><field name="VAR">martingale:tradeAgain</field>
                        <value name="VALUE"><block type="logic_boolean"><field name="BOOL">FALSE</field></block></value>
                      </block>
                    </statement>
                  </block>
                </statement>
                <statement name="ELSE">
                  <!-- profit target reached -> stop -->
                  <block type="variables_set"><field name="VAR">martingale:tradeAgain</field>
                    <value name="VALUE"><block type="logic_boolean"><field name="BOOL">FALSE</field></block></value>
                  </block>
                </statement>
              </block>
            </next>
          </block>
        </next>
      </block>
    </statement>
    <value name="RETURN"><block type="variables_get"><field name="VAR">martingale:tradeAgain</field></block></value>
  </block>

  <!-- Helper block used in trade amount cap check (returns current calculated stake) -->
  <block type="procedures_defreturn" id="get_current_stake" x="0" y="1100">
    <field name="NAME">GetCurrentStakeTemp</field>
    <statement name="STACK">
      <block type="math_arithmetic"><field name="OP">MULTIPLY</field>
        <value name="A"><block type="variables_get"><field name="VAR">martingale:multiplier</field></block></value>
        <value name="B"><block type="variables_get"><field name="VAR">martingale:initialStake</field></block></value>
      </block>
    </statement>
    <value name="RETURN"><block type="math_arithmetic"><field name="OP">MULTIPLY</field>
      <value name="A"><block type="variables_get"><field name="VAR">martingale:multiplier</field></block></value>
      <value name="B"><block type="variables_get"><field name="VAR">martingale:initialStake</field></block></value>
    </block></value>
  </block>

</xml>
