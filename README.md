# Swaptions-Payers
#Valued interest rate derivatives using QuantLib in Python, including European swaptions under the Black model by building a flat term structure, constructing vanilla interest rate swaps, and modeling implied volatility surfaces.

#Code-----

import QuantLib as ql

# Market Setup
calendar = ql.TARGET()   #Defines the calendar used for date calculations (TARGET = Eurozone calendar).Important for adjusting dates like payment and settlement.
settlement_date = ql.Date(10, 6, 2025)    #Sets the "today" or evaluation date to June 10, 2025. All pricing will be done as of this date.
ql.Settings.instance().evaluationDate = settlement_date   #Updates the global QuantLib setting to use the above settlement_date as the evaluation date.



# Flat Yield Curve
flat_rate = ql.FlatForward(settlement_date, 0.02, ql.Actual365Fixed())   #Creates a flat forward rate curve with an annualized interest rate of 2% starting from the settlement date. Actual365Fixed() specifies the day count convention.
yield_curve = ql.YieldTermStructureHandle(flat_rate) #Wraps the flat curve in a Handle, so it can be passed into pricing engines and objects like Euribor6M.

# Swaption Details
exercise_date = settlement_date + ql.Period("1Y")  # Swaption expires in 1Y. Swaption can be exercised 1 year from today.
exercise = ql.EuropeanExercise(exercise_date) # Creates a European-style exercise object (can only be exercised on the exact exercise_date).
maturity = ql.Period("5Y")  # Swap starts in 1Y and matures 5Y after that. The swap will last for 5 years if the swaption is exercised.

# Swap Schedule
start = exercise_date #The swap starts at exercise_date and ends 5 years later, adjusted for holidays using the TARGET calendar.
end = calendar.advance(start, maturity)
fixed_schedule = ql.Schedule(start, end, ql.Period("1Y"), calendar,
                              ql.ModifiedFollowing, ql.ModifiedFollowing,
                              ql.DateGeneration.Forward, False)
#Sets up the fixed leg payment schedule:
#Every 1 year. Adjusted using "Modified Following" rule. Dates generated in forward direction.  

float_schedule = fixed_schedule #(Uses the same schedule for the floating leg (simplified, though in practice it would be 6M intervals))

# Swap Legs
notional = 1_000_000 # The notional amount of the swap.
fixed_rate = 0.025 # The strike rate (fixed rate) for the swaption. This is what the option allows you to pay (or receive).
index = ql.Euribor6M(yield_curve) # Defines the floating rate index for the swap (6-month Euribor). Linked to the flat yield curve.

vanilla_swap = ql.VanillaSwap(ql.VanillaSwap.Payer,
                              notional,
                              fixed_schedule,
                              fixed_rate,
                              ql.Actual365Fixed(),
                              float_schedule,
                              index,
                              0.0,
                              ql.Actual365Fixed())

#Creates a payer swap: you pay fixed, receive floating. Uses the fixed and floating schedules. 0.0 is the spread added to the floating leg (set to zero here).

# Black Volatility and Engine
volatility = 0.20  # 20% # Assumed constant Black volatility (for the forward swap rate).
black_vol = ql.ConstantSwaptionVolatility(settlement_date, calendar,
                                          ql.ModifiedFollowing,
                                          volatility,
                                          ql.Actual365Fixed())

#Creates a flat volatility surface for swaptions. Used to model the uncertainty (vol) in forward rates.

# ✅ Fix: wrap black_vol in a handle
engine = ql.BlackSwaptionEngine(yield_curve, ql.SwaptionVolatilityStructureHandle(black_vol))
#Builds the pricing engine using the Black model. Requires: yield_curve: for discounting. black_vol: to model option volatility.

# Create Swaption and Price
swaption = ql.Swaption(vanilla_swap, exercise) # Constructs the swaption object using the defined swap and European exercise. 
swaption.setPricingEngine(engine) #Attaches the Black model pricing engine to the swaption.
# Output NPV
print(f"Swaption Price (Black model): {swaption.NPV():,.2f}")
#Calculates and prints the net present value (NPV) of the swaption — the fair price under current market assumptions.
