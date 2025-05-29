// File: com/dp/entity/Application.java
package com.dp.entity;

import lombok.Data;
import java.util.List;

@Data
public class Application {
    private String applicationId;
    private List<Transaction> transactions;
}


// File: com/dp/entity/Transaction.java
package com.dp.entity;

import lombok.Data;
import java.time.LocalDate;

@Data
public class Transaction {
    private String transactionId;
    private LocalDate date;
    private double amount;
    private String description;
    private String categoryLevelOne;
    private String categoryLevelTwo;
    private String categoryLevelThree;
}


// File: com/dp/service/Offer.java
package com.dp.service;

import lombok.Data;

@Data
public class Offer {
    private String applicationId;
    private String timestamp;
    private boolean eligibleForOffer;
    private String recommendedProduct;
    private boolean averageIncome12m;
    private boolean averageIncome6m;
    private boolean averageIncome3m;
    private boolean averageIncome1m;
    private boolean incomeMonthsCount12m;
    private boolean minimumIncome6m;
    private boolean has12MonthData;

    public Offer(String applicationId, String timestamp, boolean eligibleForOffer,
                 String recommendedProduct, boolean avg12, boolean avg6, boolean avg3, boolean avg1,
                 boolean cnt12, boolean min6, boolean has12m) {
        this.applicationId = applicationId;
        this.timestamp = timestamp;
        this.eligibleForOffer = eligibleForOffer;
        this.recommendedProduct = recommendedProduct;
        this.averageIncome12m = avg12;
        this.averageIncome6m = avg6;
        this.averageIncome3m = avg3;
        this.averageIncome1m = avg1;
        this.incomeMonthsCount12m = cnt12;
        this.minimumIncome6m = min6;
        this.has12MonthData = has12m;
    }
}


// File: com/dp/utility/IncomeUtils.java
package com.dp.utility;

import com.dp.entity.Transaction;
import java.time.YearMonth;
import java.util.*;
import java.util.stream.Collectors;

public class IncomeUtils {

    public static Map<YearMonth, Double> incomeByMonth(List<Transaction> transactions) {
        return transactions.stream()
                .filter(IncomeUtils::isIncome)
                .collect(Collectors.groupingBy(
                        tx -> YearMonth.from(tx.getDate()),
                        Collectors.summingDouble(Transaction::getAmount)
                ));
    }

    private static boolean isIncome(Transaction tx) {
        return "CREDIT".equalsIgnoreCase(tx.getCategoryLevelOne())
                && "INCOME".equalsIgnoreCase(tx.getCategoryLevelTwo())
                && ("Paychecks/Salary".equalsIgnoreCase(tx.getCategoryLevelThree())
                || "Retirement Income".equalsIgnoreCase(tx.getCategoryLevelThree()));
    }

    public static boolean averageIncomeOverThreshold(Map<YearMonth, Double> incomeMap, int months, double threshold) {
        return incomeMap.entrySet().stream()
                .sorted(Map.Entry.<YearMonth, Double>comparingByKey().reversed())
                .limit(months)
                .mapToDouble(Map.Entry::getValue)
                .average().orElse(0.0) >= threshold;
    }

    public static boolean countIncomeOverThreshold(Map<YearMonth, Double> incomeMap, int months, int minCount) {
        return incomeMap.entrySet().stream()
                .sorted(Map.Entry.<YearMonth, Double>comparingByKey().reversed())
                .limit(months)
                .filter(e -> e.getValue() > 0)
                .count() >= minCount;
    }

    public static boolean minIncomeOverThreshold(Map<YearMonth, Double> incomeMap, int months, double threshold) {
        return incomeMap.entrySet().stream()
                .sorted(Map.Entry.<YearMonth, Double>comparingByKey().reversed())
                .limit(months)
                .mapToDouble(Map.Entry::getValue)
                .min().orElse(0.0) >= threshold;
    }

    public static boolean hasDataFor12Months(List<Transaction> transactions) {
        Set<YearMonth> months = transactions.stream()
                .map(tx -> YearMonth.from(tx.getDate()))
                .collect(Collectors.toSet());
        return months.size() >= 12;
    }

    public static String recommendProduct(Map<YearMonth, Double> incomeMap) {
        double avg = incomeMap.values().stream()
                .mapToDouble(Double::doubleValue)
                .average().orElse(0.0);

        if (avg < 2000) return "Virtual Wallet Standard";
        else if (avg < 5000) return "Virtual Wallet Performance Spend";
        else return "Virtual Wallet Performance Select";
    }
}


// File: com/dp/entity/DynamicPromoService.java
package com.dp.entity;

import com.dp.service.Offer;
import com.dp.utility.IncomeUtils;

import java.time.LocalDate;
import java.time.YearMonth;
import java.util.List;
import java.util.Map;
import java.util.concurrent.CompletableFuture;

public class DynamicPromoService {

    public Offer calculateOffer(Application application) throws Exception {

        List<Transaction> transactions = application.getTransactions();

        Map<YearMonth, Double> monthlyIncome = IncomeUtils.incomeByMonth(transactions);

        CompletableFuture<Boolean> avg12m = CompletableFuture.supplyAsync(() ->
                IncomeUtils.averageIncomeOverThreshold(monthlyIncome, 12, 7500));
        CompletableFuture<Boolean> avg6m = CompletableFuture.supplyAsync(() ->
                IncomeUtils.averageIncomeOverThreshold(monthlyIncome, 6, 7500));
        CompletableFuture<Boolean> avg3m = CompletableFuture.supplyAsync(() ->
                IncomeUtils.averageIncomeOverThreshold(monthlyIncome, 3, 7500));
        CompletableFuture<Boolean> avg1m = CompletableFuture.supplyAsync(() ->
                IncomeUtils.averageIncomeOverThreshold(monthlyIncome, 1, 7500));
        CompletableFuture<Boolean> cnt12m = CompletableFuture.supplyAsync(() ->
                IncomeUtils.countIncomeOverThreshold(monthlyIncome, 12, 10));
        CompletableFuture<Boolean> min6m = CompletableFuture.supplyAsync(() ->
                IncomeUtils.minIncomeOverThreshold(monthlyIncome, 6, 5000));
        CompletableFuture<Boolean> has12MonthData = CompletableFuture.supplyAsync(() ->
                IncomeUtils.hasDataFor12Months(transactions));

        CompletableFuture.allOf(avg12m, avg6m, avg3m, avg1m, cnt12m, min6m, has12MonthData).join();

        boolean eligible = avg12m.get() && avg6m.get() && avg3m.get() && avg1m.get()
                && cnt12m.get() && min6m.get();

        String recommendedProduct = IncomeUtils.recommendProduct(monthlyIncome);

        return new Offer(
                application.getApplicationId(),
                LocalDate.now().toString(),
                eligible,
                recommendedProduct,
                avg12m.get(), avg6m.get(), avg3m.get(), avg1m.get(),
                cnt12m.get(), min6m.get(), has12MonthData.get()
        );
    }
}
